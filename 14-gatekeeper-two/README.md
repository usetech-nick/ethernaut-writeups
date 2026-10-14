# Ethernaut #14 — Gatekeeper Two

## Objective

Become the `entrant` of the `GatekeeperTwo` contract.

## Vulnerability: Exploiting Contract Creation and XOR

`GatekeeperTwo.enter(bytes8 _gateKey)` is protected by three gates:

1. `gateOne` requires `msg.sender != tx.origin`.
2. `gateTwo` requires the caller's code size to be zero.
3. `gateThree` requires the XOR of the caller's address hash and the supplied key to equal the maximum possible `uint64`.

The first gate requires an intermediary contract. The second initially seems to make that impossible, since a contract normally has code. The important detail is that a contract's runtime bytecode is not installed until its constructor finishes executing.

For the third gate, the key can be derived using XOR. XORing a value with a bitmask of all ones flips every bit, so the key can be constructed by XORing the relevant address hash with the maximum `uint64`.

## Root Cause

The contract assumes that a caller with zero code size cannot be an intermediary contract. That assumption fails during contract creation, when the constructor can make external calls before the new contract's runtime bytecode is installed.

The key check is also predictable: it uses a hash of the caller's address, and the required key can be derived from that public value.

## Attack Contract

```solidity
contract Attacker {
    GatekeeperTwo gk =
        GatekeeperTwo(0x5079C19fb9325f942e9E16c951676325e9fECf13);

    constructor() {
        bytes8 gateKey = bytes8(
            uint64(bytes8(keccak256(abi.encodePacked(address(this))))) ^
                type(uint64).max
        );
        (bool success, ) = 0x5079C19fb9325f942e9E16c951676325e9fECf13.call(
            abi.encodeWithSignature("enter(bytes8)", gateKey)
        );

        require(success, "attack failed");
    }
}
```

The constructor makes the call to `GatekeeperTwo` during deployment, before the attack contract's runtime code is installed.

For the key, I hash `address(this)`, convert the result to `bytes8`, and XOR it with `type(uint64).max`. This flips all 64 bits of the narrowed value and produces the key required by the third gate.

## Exploit

1. Read the three modifiers and identify the requirements: an intermediary caller, zero caller code size, and a key derived from the caller's address.
2. Realize that the zero-code-size condition can be satisfied by a contract calling the target from its constructor, because its runtime bytecode has not yet been installed.
3. Understand XOR: a bit XORed with `1` flips, while a bit XORed with `0` stays the same. XORing a 64-bit value with `type(uint64).max` therefore inverts every bit.
4. Use `address(this)` to derive the key from the attack contract's own address. This matters because `GatekeeperTwo` sees the attacker contract as `msg.sender`, not the EOA that deployed it.
5. Deploy the attacker contract. Its constructor calls `enter(bytes8)` on the target, satisfying the gates during deployment.
6. Verify the result by querying `entrant()` on the target contract.

The verification command returned my EOA address:

`0x0C6c78a0aee50184b3eF605225a1c605A48e6144`

This confirmed that the target's `entrant` value had been set to my address.

## Impact

An attacker can bypass the gate conditions by making the call during contract construction and deriving a valid key from the attack contract's address. The zero-code-size check does not prevent contract-based calls during deployment, and the XOR condition does not provide secrecy when the required value is derived from public information.

## Key Takeaways

- `msg.sender` is the immediate caller, while `tx.origin` is the originating EOA.
- A contract's runtime bytecode is installed only after its constructor finishes, so a call made during construction can satisfy a zero-code-size check.
- `address(this)` refers to the address of the contract being constructed, even before its runtime bytecode is installed.
- XOR with all ones flips every bit. For a 64-bit value, `type(uint64).max` is a mask of 64 ones.
- The key must be derived from the same address value that the target uses in its check.
- A successful low-level call only tells us that the call did not revert. Verifying `entrant()` confirms the resulting state.

## Mitigation

- Do not treat `extcodesize` or a zero-code-size check as proof that a caller is an EOA. Contracts can make calls during construction.
- Avoid relying on `tx.origin` for authorization. Use explicit authorization checks instead.
- Do not use predictable values derived from public addresses as secrets or authorization tokens.
- Design access control around the actual authorization requirement rather than assumptions about a caller's code size.