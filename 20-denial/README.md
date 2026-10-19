# Ethernaut #20 — Denial

## Objective

Prevent the owner from withdrawing funds from the `Denial` contract.

The challenge specifies that the attack must deny withdrawals when the transaction is executed with **1 million gas or less**.

The goal is not to steal funds. It is to cause a denial of service (DoS) by making the owner's `withdraw()` function fail.

## Vulnerability

The contract allows an arbitrary address to become the withdrawal partner:

```solidity
function setWithdrawPartner(address _partner) public {
    partner = _partner;
}
```

During a withdrawal, the contract sends 1% of its ETH balance to the partner using a low-level call:

```solidity
uint256 amountToSend = address(this).balance / 100;

partner.call{value: amountToSend}("");
payable(owner).transfer(amountToSend);
timeLastWithdrawn = block.timestamp;
withdrawPartnerBalances[partner] += amountToSend;
```

The partner call forwards most of the available gas to the recipient. Because the partner address is user-controlled, an attacker can register a malicious contract whose `receive()` function deliberately consumes all the gas forwarded to it.

Although the low-level call's return value is ignored, the withdrawal can still fail if the caller has insufficient gas remaining to complete its subsequent operations.

## Root Cause

The vulnerability arises from the combination of:

- **Untrusted external interaction:** Anyone can set the withdrawal partner.
- **Unbounded gas consumption:** The partner can execute arbitrary code using the gas forwarded by `call`.
- **Insufficient gas reserve:** The withdrawal function needs gas after the partner call to pay the owner and update storage.
- **No effective isolation:** The partner's execution can exhaust the gas budget available to the overall withdrawal.

Ignoring the return value of `call()` does not prevent a denial of service. It only means that a failed partner call does not automatically revert the caller.

## Exploit

### 1. Create a malicious partner

The attack contract deliberately enters an infinite loop when it receives ETH.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Attack {

    receive() external payable {
        while (true) {}
    }
}
```

The loop never completes normally. It keeps consuming gas until the partner call runs out of gas.

The increment is not necessary for the attack; an empty infinite loop can also consume the forwarded gas.

### 2. Deploy the attack contract

Deploy the `Attack` contract and retain its address.

### 3. Register it as the partner

Call `setWithdrawPartner()` on the target `Denial` instance, passing the deployed attack contract's address.

Once registered, the malicious contract receives the partner payout whenever `withdraw()` is called.

### 4. Trigger the withdrawal

When `withdraw()` executes:

1. The target calculates the partner payout.
2. It calls the malicious partner and forwards most of the available gas.
3. The attack contract enters its infinite loop and consumes the forwarded gas.
4. EIP-150 leaves approximately 1/64 of the caller's available gas outside the subcall.
5. Under the challenge's 1-million-gas limit, that reserve is insufficient to finish the remaining withdrawal operations.
6. The overall transaction runs out of gas and reverts.

Because the entire transaction reverts, the owner's ETH transfer and the target's state changes are rolled back too.

## Investigation and Testing

I initially investigated the attack by testing different transaction gas limits locally with Foundry's `cast` and inspecting execution traces.

| Experiment | Observation |
|---|---|
| Default `cast send` gas estimation | Transaction succeeded with approximately 14.9M gas used |
| Explicit 230,000 gas limit | Transaction reverted with out-of-gas |
| Explicit 2,300,000 gas limit | Transaction reverted with out-of-gas, despite reaching the owner's transfer during execution |

Initially, I focused on why the transaction succeeded with a large gas budget but reverted with smaller limits. I later revisited the challenge specification and identified the key constraint: **the owner must be unable to withdraw when the transaction has 1M gas or less.**

This clarified the exploit. The malicious partner consumes the gas forwarded to it, leaving the withdrawal function with insufficient gas to finish its remaining operations under the specified constraint.

These experiments helped me understand the interaction between gas forwarding, EIP-150, and transaction-wide rollback. They also reinforced an important testing lesson: execution traces and gas-limit experiments must be interpreted in the context of the actual security requirement.

## Impact

An attacker can prevent the owner from withdrawing funds under the specified gas constraint.

This is a **denial-of-service vulnerability** affecting the availability of the withdrawal functionality. It does not require the attacker to steal the contract's ETH.

## Key Takeaways

- A low-level `call()` returning `false` does not automatically revert its caller.
- A subcall running out of gas does not necessarily mean the parent transaction will fail.
- EIP-150 limits forwarded gas to approximately 63/64 of the caller's available gas, leaving a reserve for subsequent operations.
- That reserve is not necessarily sufficient to complete the parent function.
- Gas limits matter: a gas-griefing attack can succeed under a constrained budget but fail to deny service when more gas is available.
- Gas estimation is not a substitute for testing the actual gas constraints imposed by a challenge or production system.
- Denial of service is a security impact even when no funds are stolen.

## Mitigation

A safer design should avoid making a critical withdrawal depend on arbitrary partner-controlled execution.

Possible mitigations include:

1. **Use pull payments:** Record amounts owed to recipients and allow them to withdraw independently, so a failing recipient cannot block the owner's withdrawal.
2. **Bound gas forwarded to external calls:** If the partner interaction must remain, apply an intentional gas limit and design the failure path carefully.
3. **Separate critical operations:** Ensure that a partner's execution cannot prevent the owner from receiving funds or block essential state updates.
4. **Validate partner configuration:** Restrict who can change the partner and apply appropriate access controls.

Simply ignoring the return value of an external call is not enough to make the withdrawal flow resistant to gas-based denial of service.

## Conclusion

Ethernaut #20 demonstrated how an attacker-controlled external call can consume the gas required by a critical function.

The key was not to steal funds or make the partner call revert in isolation. It was to exhaust the available gas so that `withdraw()` could not finish within the challenge's specified gas limit.

**Challenge #20 — Denial: Solved.**