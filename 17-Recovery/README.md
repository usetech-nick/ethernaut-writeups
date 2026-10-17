# Ethernaut #17 — Recovery

## Objective

Recover the ETH held by a lost `SimpleToken` contract created by the `Recovery` factory.

## Vulnerability / Concept

Contract addresses created with `CREATE` are deterministic. Even when a factory doesn't store the address of a newly deployed contract, that contract can still be located using on-chain information.

## Root Cause

The `Recovery` contract deploys tokens using `new SimpleToken(...)` but doesn't save the resulting address in a state variable or emit an event containing it.

The deployed token still exists on-chain and can be interacted with directly if its address is recovered.

## Exploit

1. Inspected the `Recovery` and `SimpleToken` contracts to understand the factory deployment and token behavior.
2. Opened the factory's transaction history on Etherscan and inspected its internal transactions.
3. Found the address of the deployed `SimpleToken` contract holding ETH.
4. Called `destroy(address)` directly on the recovered contract, specifying my EOA as the recipient.

## Proof of Completion

- **Recovered contract:** `0x607f35861a8234A06578c05658B56Bd69a3F1103`
- **Recipient:** `0x0C6c78a0aee50184b3eF605225a1c605A48e6144`
- **Transaction:** `0xc433d1db2c2a208930a0e95d5af936a277bf02b2792a0c1658c10635ae6eb179`
- **Status:** Success

Executed the call directly from the terminal using Foundry's `cast send` and my local keystore.

## Key Takeaways

- A factory doesn't need to store a deployed contract's address for that contract to remain accessible.
- `CREATE` addresses are deterministic, based on the creator address and its creation nonce.
- Block explorers can expose internal contract creations, making them useful for on-chain investigations.
- Not every challenge requires an attack contract; sometimes tracing on-chain activity is enough.

## Modern Ethereum Note

Under EIP-6780, `SELFDESTRUCT` generally no longer deletes a contract's code and storage unless the contract was created and destroyed in the same transaction. It can still transfer the contract's ETH balance to the specified recipient.

## Final Lesson

Losing a contract address doesn't make the contract disappear. Learn to investigate deployment history and interact with contracts directly.