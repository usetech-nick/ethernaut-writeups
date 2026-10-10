# Ethernaut #3 — Fallout

## Vulnerability: Incorrect Constructor Naming

### The Catch

The contract uses Solidity `0.6.0`, where constructors were declared using the contract's name. But the supposed constructor is named `Fal1out()` instead of `Fallout()` — a `1` instead of an `l`.

Because the names don't match, Solidity treats `Fal1out()` as a regular public function rather than a constructor.

### Exploit

```javascript
await contract.Fal1out()
```

That's it. One call, and ownership is transferred to the caller.

### Root Cause

The function assigns `msg.sender` to `owner` without any access control. Since it's publicly callable, anyone can overwrite the current owner's address.

The function is also `payable`, but sending ETH isn't necessary because it never checks `msg.value`.

### Impact

An attacker can take ownership of the contract and gain access to owner-restricted functionality.

### Mitigation

Use Solidity's `constructor` keyword instead of naming the constructor after the contract. Modern Solidity makes this distinction explicit and prevents this particular naming mistake.

### Key Takeaway

**Never assume a function is a constructor just because its name looks like one.** Verify how the compiler interprets it, especially when reviewing older Solidity contracts.