# Ethernaut Writeups

My solutions and technical writeups for [Ethernaut](https://ethernaut.openzeppelin.com/), OpenZeppelin's smart contract security wargame.

The goal is to understand smart contract vulnerabilities, reproduce exploits, and document their root causes, impact, and mitigations.

## Progress

| # | Challenge | Vulnerability / Concept | Status |
|---|---|---|---|
| 0 | Hello Ethernaut | Introduction to the platform | ✅ Completed (tutorial, no writeup) |
| 1 | [Fallback](./01-fallback/) | Improper access control, unauthorized ownership transfer | ✅ Solved |
| 2 | [Fallout](./02-fallout/) | Incorrect constructor naming | ✅ Solved |
| 3 | [Coin Flip](./03-coinflip/) | Predictable on-chain randomness | ✅ Solved |
| 4 | [Telephone](./04-telephone/) | `tx.origin` vs `msg.sender` | ✅ Solved |
| 5 | [Token](./05-token/) | Integer underflow | ✅ Solved |
| 6 | [Delegation](./06-delegate/) | Unsafe `delegatecall` with user-controlled calldata | ✅ Solved |
| 7 | [Force](./07-force/) | Forced ether transfer via `selfdestruct` | ✅ Solved |
| 8 | [Vault](./08-vault/) | Reading "private" storage on-chain | ✅ Solved |

## Writeup Structure

Each documented challenge covers:

- **Vulnerability:** What is wrong with the contract?
- **Root Cause:** Why does the vulnerability exist?
- **Exploit:** Steps to reproduce the attack.
- **Impact:** What can an attacker achieve?
- **Mitigation:** How could the issue be prevented?
- **Proof of Concept:** Foundry tests where applicable.

## Tools

- Solidity
- Foundry
- Ethereum browser console
- Git and GitHub

## Goal

Build practical smart contract security skills through hands-on exploitation, vulnerability analysis, and reproducible testing.

---

*Learning smart contract security one vulnerability at a time.*