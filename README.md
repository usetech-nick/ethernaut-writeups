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
| 7 | [Force](./07-force/) | Forced Ether transfer via `selfdestruct` | ✅ Solved |
| 8 | [Vault](./08-vault/) | Reading "private" storage on-chain | ✅ Solved |
| 9 | [King](./09-king/) | Denial of service via failed ETH refund | ✅ Solved |
| 10 | [Re-entrancy](./10-reentrancy/) | Reentrancy via state update after external call | ✅ Solved |
| 11 | [Elevator](./11-elevator/) | Trusting an external contract to answer consistently | ✅ Solved |
| 12 | [Privacy](./12-privacy/) | Reading private storage, storage packing | ✅ Solved |
| 13 | [Gatekeeper One](./13-gatekeeper-one/) | Gas-sensitive access control, key construction | ✅ Solved |
| 14 | [Gatekeeper Two](./14-gatekeeper-two/) | Constructor-time calls, code-size checks, XOR | ✅ Solved |
| 15 | [Naught Coin](./15-naught-coin/) | Bypassing a transfer restriction via `transferFrom()` | ✅ Solved |
| 16 | [Preservation](./16-preservation/) | `delegatecall` storage collision, library address takeover | ✅ Solved |
| 17 | [Recovery](./17-recovery/) | Recovering a lost contract address, deterministic `CREATE` addresses | ✅ Solved |

## Writeup Structure

Each documented challenge covers:

- **Vulnerability:** What is wrong with the contract?
- **Root Cause:** Why does the vulnerability exist?
- **Exploit:** Steps to reproduce the attack.
- **Impact:** What can an attacker achieve?
- **Mitigation:** How could the issue be prevented?
- **Proof of Concept:** Attack contracts and Foundry tests where applicable.

## Tools

- Solidity
- Foundry
- Ethereum browser console
- Git and GitHub

## Goal

Build practical smart contract security skills through hands-on exploitation, vulnerability analysis, and reproducible testing.

---

*Learning smart contract security one vulnerability at a time.*
