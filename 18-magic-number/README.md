# Ethernaut #18 — MagicNumber

## Objective

Provide a `Solver` contract whose runtime code is at most 10 bytes and that returns `42` when called with `whatIsTheMeaningOfLife()`.

## Vulnerability / Concept

This level isn't a vulnerability; it's an exercise in how contract deployment works at the EVM level. A contract's code is whatever bytes the initialization code `RETURN`s, so the runtime code can be hand-written in raw opcodes instead of compiled from Solidity.

## Root Cause

Solidity-compiled contracts include a dispatcher, a free-memory-pointer preamble and metadata, so even a trivial function produces far more than 10 bytes. The size limit can only be met by skipping the compiler and writing the bytecode directly.

## Exploit

1. Wrote the runtime code: store `42` in memory, then return that 32-byte word.
   - `PUSH1 0x2a; PUSH1 0x00; MSTORE; PUSH1 0x20; PUSH1 0x00; RETURN`
   - Bytecode: `602a60005260206000f3` (10 bytes). It has no function dispatch, so it returns `42` for any call.
2. Wrote the init code that places the runtime bytes in memory and returns them.
   - `PUSH10 <runtime>; PUSH1 0x00; MSTORE; PUSH1 0x0a; PUSH1 0x16; RETURN`
   - `MSTORE` right-aligns the 10 bytes in a 32-byte word, so they start at offset 22 (`0x16`) with size 10 (`0x0a`).
   - Bytecode: `69602a60005260206000f3600052600a6016f3` (19 bytes).
3. Deployed a `Factory` contract whose constructor runs `create` with the init code (size `0x13`) and calls `setSolver` on the level instance with the new address.
4. Fetched the code at the deployed solver with `cast code` and confirmed it matched the 10-byte runtime exactly, then submitted the instance.

## Proof of Completion

- **Factory:** `0x384E62F1254d76CcC59B90435cC7d01baeD71275`
- **Solver (created by the factory):** `0x316ac41F69CCB1FE7C3da9F0E231Dc436c6F2aD8`
- **Deployer:** `0x0C6c78a0aee50184b3eF605225a1c605A48e6144`
- **Solver code:** `0x602a60005260206000f3`
- **Level instance:** `0x3FBbd304D619ea90539923452AcDF913dC38Cdc4`
- **Transaction:** `0xe83c15bcc6946f62c584098e98872a4760d25700480b2ba86b988bc1f787325e`
- **Status:** Success

Deployed the factory with Foundry's `forge create` using my local keystore.

## Key Takeaways

- Init code runs once at deployment; the bytes it `RETURN`s become the contract's runtime code.
- `MSTORE` writes a full 32-byte word and right-aligns shorter values, which determines the offset passed to `RETURN`.
- The factory's address is not the address of the contract it creates. Always check the code at the address you expect.
- `cast call` only simulates; state-changing calls need `cast send`.
- `PUSH1` consumes one data byte after the opcode, so bytecode is opcodes and data bytes laid out back to back.

## Modern Ethereum Note

Since the Shanghai upgrade (EIP-3855), the `PUSH0` opcode (`5f`) pushes zero in a single byte. Replacing both `PUSH1 0x00` instructions in the runtime shortens it to 8 bytes (`602a5f5260205ff3`), but the 10-byte version works on any chain.

## Final Lesson

A deployed contract is just bytes that some init code chose to return. Once you understand that, the compiler stops being a black box.