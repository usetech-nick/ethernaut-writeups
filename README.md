# Ethernaut Writeups

My solutions and technical writeups for [Ethernaut](https://ethernaut.openzeppelin.com/), OpenZeppelin's smart contract security wargame.

The goal is to understand smart contract vulnerabilities, reproduce exploits, and document their root causes, impact, and mitigations.

## Progress

| # | Challenge | Vulnerability / Concept | Status |
|---|---|---|---|
| 1 | Hello Ethernaut | Introduction to the platform | ✅ Completed (tutorial, no writeup) |
| 2 | [Fallback](./02-fallback/) | Improper access control, unauthorized ownership transfer | ✅ Solved |
| 3 | [Fallout](./03-fallout/) | Incorrect constructor naming | ✅ Solved |
| 4 | [Coin Flip](./04-coin-flip/) | Predictable on-chain randomness | ✅ Solved |

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