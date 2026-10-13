# Ethernaut #12 — Privacy

## Objective

Unlock the contract by making `locked` `false`.

## Vulnerability: Secrets Stored in "Private" Contract Storage

`unlock(bytes16 _key)` only succeeds if `_key` matches the first 16 bytes of `data[2]`, a `private` variable. As in Vault, the developer treated `private` as if it meant secret. It only stops other contracts from reading the variable through Solidity. The value still sits in a storage slot that anyone can read from a node.

What makes this level harder than Vault is finding the right slot, because the layout involves packing and an array.

## Root Cause

Confusing language-level visibility with confidentiality, as in Vault. Storage is public, and its layout is deterministic, so anyone can compute where each variable lives.

## Storage Layout

| Slot | Variable | Notes |
|---|---|---|
| 0 | `locked` (`bool`) | 1 byte. The next variable is a `uint256` and doesn't fit in the remaining 31 bytes, so it starts a new slot. |
| 1 | `ID` (`uint256`) | Takes a full slot. |
| 2 | `flattening` (`uint8`), `denomination` (`uint8`), `awkwardness` (`uint16`) | 1 + 1 + 2 = 4 bytes, packed together into one slot. |
| 3 | `data[0]` | A static array of `bytes32` doesn't pack. It starts a new slot, and each element takes a full slot. |
| 4 | `data[1]` | |
| 5 | `data[2]` | The one `unlock` compares against. |

## Exploit

1. Work out the layout above and identify `data[2]` as slot 5. Only that slot matters, since `unlock` compares against `data[2]` alone.
2. Read slot 5 from the instance: `cast storage <instance-address> 5 --rpc-url $SEPOLIA_RPC_URL`. This returned the full 32-byte value of `data[2]`:
   `0x8b2f88b32e31654a5d884dbfacaedfc95016a5b2c09638bfa7d338f7b975fe32`
3. Take the first 16 bytes. `bytes16(data[2])` keeps the **left** half, which is `0x` plus the first 32 hex characters (`cut -c1-34`):
   - kept (the key): `0x8b2f88b32e31654a5d884dbfacaedfc9`
   - discarded: `5016a5b2c09638bfa7d338f7b975fe32`4. Call `unlock` with that key: `cast send <instance-address> "unlock(bytes16)" 0x8b2f88b32e31654a5d884dbfacaedfc9 --rpc-url $SEPOLIA_RPC_URL --account <keystore name>`. The transaction succeeded.
5. Verify with `cast call <instance-address> "locked()(bool)" --rpc-url $SEPOLIA_RPC_URL`, which returned `false`, then submit.

**On the truncation direction.** Converting a `bytes32` to `bytes16` keeps the *first* (leftmost) 16 bytes, while converting between integer types keeps the *low-order* bits. Taking the wrong half is the easiest way to get a key that looks right and fails, and it's the part I'd check first if the call reverts.

## Impact

Anything placed in contract storage can be read by anyone, regardless of visibility keywords. A key, password or secret stored this way is exposed from the moment it is written.

## Key Takeaways

- `private` is not private on-chain. It restricts access from other contracts through Solidity, not access to the data.
- Small variables pack into shared slots when they fit, and a variable that doesn't fit starts a new slot.
- Static arrays don't pack their elements: each `bytes32` takes a whole slot.
- Slot numbers follow declaration order, so reading a contract's storage is mostly careful counting.
- `bytes32` to `bytes16` keeps the left half. Integer narrowing keeps the right.

## Mitigation

- Never store secrets on-chain in plaintext, whatever their visibility.
- Keep secrets off-chain and store only a hash on-chain, so the contract can verify without revealing anything. If the secret is guessable, a hash can still be brute-forced.
- Don't use on-chain "keys" to gate state changes without something an attacker can't read, such as a signature from a key held off-chain.