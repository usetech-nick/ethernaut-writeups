# Ethernaut #8 — Vault

## Objective
Unlock the vault by finding the password.

## Vulnerability: Sensitive Data Stored On-Chain

`Vault` stores a password in a `private` variable and only unlocks when `unlock` is called with that password. The developer assumed `private` meant the value couldn't be read.

It doesn't. In Solidity, `private` only stops *other contracts* from accessing the variable through the language. The value itself still lives in the contract's storage, and storage on a public blockchain is readable by anyone.

## Root Cause

Confusing language-level visibility with data confidentiality. `private` controls which code can reference a variable. It does nothing to hide the underlying storage slot, which anyone can read directly from a node.

## Exploit

1. Read the contract and note the state variables in declaration order: a `bool locked` and a `bytes32 password`.
2. Work out which storage slot holds the password. Slots are numbered from 0 in declaration order, and consecutive variables share a slot only if the next one fits in the space left. `locked` is a `bool` (1 byte), but `password` is a `bytes32` (a full 32 bytes), so it can't fit alongside it. That puts `locked` in slot 0 and `password` in slot 1.
3. Read slot 1 straight from the node, using the standard `eth_getStorageAt` RPC call. The result is the 32-byte value as hex.
4. Call `unlock(bytes32)` with that hex value. No conversion is needed, because the function takes the raw `bytes32`.
5. Check that `locked` is now `false`, then submit the instance.

Decoding the hex as ASCII shows the password was a readable string, which is a fitting joke for a "very strong secret password". It isn't needed for the attack, since the function takes the raw bytes.

## Impact

Anything stored in contract storage can be read by anyone, regardless of visibility keywords. Passwords, keys, and any "hidden" game state are exposed the moment they're written.

## Key Takeaways

- `private` is not private on-chain. It only restricts access from other contracts through Solidity.
- All contract storage is public and readable at any time.
- Storage layout is deterministic, so anyone can compute which slot holds which variable, including how smaller types get packed together.

## Mitigation

- Never store secrets on-chain in plaintext, whatever visibility they carry.
- Keep secrets and encryption off-chain, and store only a hash of the result on-chain, so the contract can verify without revealing anything.
- Be careful with the hash approach: if the secret is guessable, such as a short password, a hash can still be brute-forced from the stored value, and revealing the original value in a transaction exposes it to everyone.