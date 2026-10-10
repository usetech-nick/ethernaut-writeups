# Ethernaut #2 — Fallback

## Vulnerability: Improper Access Control

### The Catch

The real catch is that we only need to send a tiny amount of ETH to become the owner. We don't need to contribute more ETH than the current owner.

The `receive()` function checks two conditions:
- The caller must have contributed before (`contributions[msg.sender] > 0`).
- The caller must send ETH with the transaction (`msg.value > 0`).

Both conditions are trivial to satisfy.

### Attack Steps

1. Call `contribute()` with literally **1 wei**.
2. Now we're a contributor, satisfying the first condition in `receive()`.
3. Send another **1 wei** directly to the contract address. Both conditions are fulfilled, and ownership transfers to us.
4. Call `withdraw()` and drain the contract!

### Root Cause

The contract transfers ownership to anyone who has previously contributed and sends ETH through `receive()`. Since contributing just 1 wei is enough to satisfy the requirement, an attacker can take ownership without outbidding the current owner.

### Impact

An attacker can gain ownership of the contract and withdraw its entire ETH balance.

### Mitigation

Ownership transfers should not be triggered by ordinary ETH transfers. Use explicit, properly authorized ownership-transfer mechanisms, such as OpenZeppelin's `Ownable` with its `onlyOwner` and `transferOwnership` functionality.

### Key Takeaway

**Never use trivial user interactions as authorization for privileged actions.** A condition being technically satisfied doesn't mean it provides meaningful security.