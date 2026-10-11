# Ethernaut #7 — Delegation

## Objective
Claim ownership of the `Delegation` contract instance.

## Vulnerability: Unsafe `delegatecall` with User-Controlled Calldata

`Delegation` forwards any call it doesn't recognise to `Delegate` through its fallback:

`address(delegate).delegatecall(msg.data);`

Two things combine to make this dangerous:

1. **`delegatecall` runs the target's code in the caller's storage context.** The code comes from `Delegate`, but it reads and writes `Delegation`'s storage. `msg.sender` is also preserved, so it is still the original caller.
2. **`msg.data` is fully attacker-controlled.** Anyone can make `Delegation` run any function of `Delegate`.

`Delegate.pwn()` sets `owner = msg.sender`. Both contracts declare `owner` as their first variable, so it sits in slot 0 in each. When `pwn()` runs via `delegatecall`, it overwrites slot 0 of `Delegation`, which is `Delegation`'s `owner`.

## Exploit

1. Read the `Delegation` fallback and notice it forwards raw `msg.data` through `delegatecall`.
2. Notice that `Delegate.pwn()` assigns `owner = msg.sender` and that both contracts store `owner` in slot 0.
3. Compute the selector for `pwn()` with Foundry: `cast sig "pwn()"` returns `0xdd365b8b`.
4. Send a transaction from my own account to the `Delegation` instance with that selector as calldata:

   `await contract.sendTransaction({ data: "0xdd365b8b" })`

5. `Delegation` has no `pwn()`, so the call lands in the fallback and gets forwarded.
6. Verify with `await contract.owner()` that the owner is now my address, then submit the instance.

**Note on my first attempt:** Coming from Foundry, I assumed I needed a script to make the call, so I wrote a helper contract in Remix to call `pwn()` on the instance. That was the wrong tool. Remix deploys a contract, so `msg.sender` inside the delegatecall would have been the helper contract, not my account, and I can't use Foundry cheatcodes like `vm.prank` there to change that. The call has to come straight from my EOA, and the Ethernaut console can send raw calldata directly.

## Key Takeaways

- `delegatecall` executes foreign code against your own storage.
- `msg.sender` is preserved across a `delegatecall`, which is why the caller became the owner.
- Storage layout must match between a proxy and its implementation, otherwise one contract's variables are overwritten by the other's writes.
- A fallback that forwards arbitrary calldata exposes every function of the target, including ones never meant to be reachable this way.
- Knowing how to send raw calldata from a plain EOA is a practical skill. Not every exploit needs an attacker contract.

## Prevention

- Do not forward arbitrary `msg.data` through `delegatecall`. Restrict the callable functions to an explicit allowlist of selectors.
- If delegation is genuinely needed to change ownership, put proper access control on those calls so only authorised accounts can trigger them.
- Use audited implementations such as OpenZeppelin's proxy contracts instead of a hand-rolled fallback, and follow a storage layout convention that avoids slot collisions.