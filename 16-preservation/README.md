# Ethernaut #16 — Preservation

## Objective

Become the owner of the `Preservation` contract by exploiting its use of `delegatecall`.

## Vulnerability

The contract delegates execution to external library addresses stored in its own storage. Because `delegatecall` executes the library's code in the caller's storage context, a library function can modify variables belonging to `Preservation`.

The problem becomes exploitable when the library's storage layout differs from the target's intended layout.

## Root Cause

`Preservation` stores two library addresses before its `owner` variable:

| Storage slot | `Preservation` variable |
|---|---|
| Slot 0 | `timeZone1Library` |
| Slot 1 | `timeZone2Library` |
| Slot 2 | `owner` |
| Slot 3 | `storedTime` |

The library contract declares `storedTime` as its first state variable, so its code writes to slot 0.

When invoked through `delegatecall`, that write affects **slot 0 of `Preservation`**, which contains `timeZone1Library`, rather than a separate library storage slot.

The contract also allows the address stored in slot 0 to determine the destination of a subsequent delegated call.

## Exploit

### 1. Create a malicious implementation

I deployed a minimal contract with a compatible `setTime(uint256)` function. I deliberately placed two address variables before my `owner` variable to align it with slot 2 in `Preservation`.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Attacker {
    address private __gap_slot_0;
    address private __gap_slot_1;

    uint256 public owner;

    function setTime(uint256 _time) external {
        owner = _time;
    }
}
```

The `owner` variable is a `uint256` here, but that does not prevent its storage write from overwriting the target's `address owner` at the same slot.

### 2. Overwrite the first library address

The first call to `setFirstTime()` delegates to the original library.

Passing the numeric representation of my implementation's address as the timestamp causes the library's `setTime()` function to write that value into slot 0 of `Preservation`.

The first library address is now replaced with my implementation's address.

### 3. Execute my implementation through `delegatecall`

I call `setFirstTime()` again.

This time, `Preservation` delegates execution to my implementation. Its `setTime(uint256)` function writes the supplied value to slot 2, which is the target's `owner` slot.

Passing my EOA address as a numeric value therefore changes the target's owner.

### 4. Deployment

Implementation contract deployed on Sepolia:

- **Deployer:** `0x0C6c78a0aee50184b3eF605225a1c605A48e6144`
- **Implementation:** `0xb3A9Dabea4C23D2EBb7215633e6c8d08f7919748`
- **Deployment transaction:** `0x0fa78ec2eb4b055584469c6a5a2fb72b64abe86c420e3e810147daedb624ae4a`

I triggered the exploit through terminal calls rather than adding a separate attack function to the implementation contract.

## Impact

An attacker can overwrite the library address and redirect subsequent delegated execution to attacker-controlled code. Since that code runs in the target's storage context, it can overwrite sensitive state, including ownership.

## Key Takeaways

- `delegatecall` executes external code against the caller's storage.
- Storage layout is positional: variable names and types do not protect a storage slot.
- A storage collision can corrupt a function's future execution destination.
- Function selectors and argument encoding must match the way the target invokes the delegated code.
- Storage gaps, which I had previously learned about while studying proxy upgradeability in Cyfrin Updraft, helped me reason about the slot alignment required for this exploit.

## Mitigation

- Avoid arbitrary or unsafe delegated execution.
- Ensure implementation and proxy storage layouts remain compatible.
- Treat delegatecall target addresses as security-critical state.
- Use established proxy patterns and validate storage layout across upgrades.

## Final Lesson

The same storage-layout principles used to build upgradeable systems can help uncover serious vulnerabilities in them. Understanding how `delegatecall` maps code to storage is essential for both protocol development and security research.