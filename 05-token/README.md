# Ethernaut #5 — Token

## Objective
Hack the token contract and obtain more tokens than you initially had.

## Vulnerability: Integer Underflow

The contract uses Solidity `^0.6.0`, where unsigned integer arithmetic can overflow or underflow without automatically reverting.

The vulnerable check is:

`require(balances[msg.sender] - _value >= 0);`

At first glance, this looks like a valid balance check. However, the subtraction happens **before** the comparison.

Since `balances[msg.sender]` is a `uint256`, it cannot represent negative numbers. If the sender has 20 tokens and attempts to transfer 21, the subtraction underflows and wraps around to a very large unsigned integer.

That result is always greater than or equal to zero, so the `require` condition passes.

The subsequent subtraction also underflows, leaving the sender with an enormous token balance.

## Exploit

1. Inspect the `transfer()` function and identify the flawed balance check.
2. Recognize that subtracting before comparing allows an underflow to bypass the check.
3. Call `transfer()` with a value greater than the account's current balance.
4. Verify that the account's balance has increased to an unexpectedly large value.
5. Submit the instance to complete the level.

## Key Takeaways

- Unsigned integers cannot represent negative numbers.
- In Solidity 0.6, arithmetic underflow wraps around unless explicitly checked.
- A condition intended to prevent underflow can itself trigger underflow.
- Solidity 0.8 and later versions revert on arithmetic underflow by default, unless checked arithmetic is explicitly bypassed.
- Security reviews should examine not only what a condition checks, but also **how its expression is evaluated**.

## Prevention

Compare the balance directly against the transfer amount before performing subtraction:

`require(balances[msg.sender] >= _value);`

This avoids performing the unsafe subtraction inside the validation condition.
