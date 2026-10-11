# Ethernaut #7 — Force

## Objective
Make the balance of the `Force` contract greater than zero.

## Vulnerability: Forced Ether Transfer via `selfdestruct`

`Force` is an empty contract. It has no functions, no `receive`, and no payable fallback, so every ordinary way of sending it ETH (`transfer`, `send`, a call with value, or a plain wallet transfer) reverts.

However, ordinary transfers are not the only way ETH can reach an address. `selfdestruct(target)` sends the destroying contract's entire balance to `target` at the EVM level, and **the target's code never runs**. No `receive`, no fallback, and no `require` can reject it.

## Root Cause

The contract's design assumes it controls every way value can arrive. Not having a payable function only blocks the *normal* routes. A balance can still be increased by `selfdestruct`, and by coinbase rewards or pre-funding an address before a contract is deployed to it.

## Exploit

1. Read `Force` and notice it has nothing to call and nothing that accepts ETH.
2. Ask a different question: how can ETH reach an address that doesn't want it? Ordinary transfers always run the receiver's code, so the answer has to be something that skips it.
3. Recognise that `selfdestruct` does exactly that.
4. Write an attacker contract that self-destructs to the `Force` instance:

```solidity
contract ForceAttack {
    constructor(address payable target) payable {
        selfdestruct(target);
    }
}
```

5. Deploy it with a small amount of ETH attached (1 wei is enough), passing the `Force` instance as the target. The destruction happens during deployment, so one transaction does the whole attack. (See the practical note below for why I chose this over a two-step version.)
6. Check that the instance's balance is now above zero, then submit the instance.

**Note on `selfdestruct` after Cancun (EIP-6780):** since the Cancun upgrade, `selfdestruct` only deletes a contract if it was created in the same transaction. The ETH transfer to the target still works in every case, so the attack is unaffected. The attacker contract self-destructing in its own constructor is also the one case where the code still gets removed.

## Practical Note: Why One Transaction Instead of Two

My first version used two transactions: deploy the attacker, then call an `attack(address)` function that held the `selfdestruct`. The second transaction was rejected by the RPC node with:

`in-flight transaction limit reached for delegated accounts`

The cause was my testnet account. It had an EIP-7702 delegation set on it (its code started with `0xef0100` followed by a delegate address), which looked like MetaMask's smart-account implementation. Nodes limit delegated accounts to one pending transaction at a time, so sending two back to back failed, even when waiting for the first to confirm.

Moving the `selfdestruct` into the constructor made the whole attack a single transaction, which sidestepped the limit.

If you hit the same error:

- Check whether your account has delegated code: `eth_getCode` on your own address, and look for the `0xef0100` prefix.
- Either collapse the attack into one transaction, switch the account back to a standard one, or use a fresh account that was never upgraded.

## Impact

An attacker can force ETH into any contract, whatever its code says. For `Force` that is harmless. For a contract whose logic depends on its own balance, it can break accounting, skew calculations, or lock a function permanently.

## Key Takeaways

- A contract cannot refuse ETH sent via `selfdestruct`. Lacking a payable function only blocks the normal routes.
- `address(this).balance` is not a safe source of truth, because anyone can increase it from outside.
- The question "who controls every way value can reach this contract?" is worth asking about every contract that holds funds.

## Mitigation

- Never rely on `address(this).balance == 0` (or on any exact balance) as an invariant for contract logic.
- Track deposits with an internal accounting variable that only your own functions update, and use that value for your logic instead of the raw balance.
- Treat any surplus balance as untracked donations, so unexpected ETH can't affect behaviour.