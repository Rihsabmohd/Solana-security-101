# Solana Security 101

Security remains one of the most challenging aspects of Solana program development.  
Because **small mistakes in account handling, authority checks, and CPI usage can lead to catastrophic failures**.

This repository is an **educational security reference** for Solana developers.  
It focuses on **common, real-world vulnerability patterns**, showing how they occur and how to fix them correctly.

# Prerequisites

This repository focuses on Solana security patterns, not beginner Solana development. To follow along comfortably, you should have:

Basic experience writing Solana programs in Rust

A working understanding of the Solana account model (accounts, ownership, signers, PDAs)

Familiarity with Rust fundamentals and error handling

Prior exposure to Anchor or native Solana programs (helpful, but not required)

You do not need prior auditing or security experience.
The examples are designed to build security intuition through practical, real-world patterns.

---

## Why Solana Security Is Different

Solana’s programming model is fundamentally different from EVM-based chains.

On Solana:
- Programs do **not** implicitly own their storage
- All accounts are passed in as **instruction inputs**
- Every account is **user-controlled unless explicitly validated**

This means that **account validation is the primary security boundary**.

Most Solana exploits are as a result of:
- Missing account ownership checks
- Incorrect authority assumptions
- Unsafe CPI behavior
- Arithmetic edge cases
- Improper state initialization

Understanding the Solana account model is therefore essential to writing secure programs.

---

## A Brief Introduction to Solana Accounts

A Solana account consists of:
- A public key
- An owner (program ID)
- Lamports (SOL balance)
- Arbitrary data

Important properties:
- Programs can **only modify accounts they own**
- Programs **must not assume** relationships between accounts
- Signer status only proves a key signed the transaction — **not that it is authorized**

> **Mental model:**  
> Every account is untrusted input unless your program proves otherwise.

This repository is built around that principle.

---

## Solana Security in Practice

Solana security is mostly about enforcing **intent**:
- Who is allowed to do this?
- Which accounts are valid?
- What invariants must always hold?

If these rules are not enforced explicitly in code, an attacker will enforce their own.

---

## Frameworks: Anchor and Pinocchio
For this repo we ultimately expect you to have basic fluency in writing rust, and also be familiar with the popular framewrorks for writing on solana

### Anchor

Anchor is the most widely used framework for Solana development.

It provides:
- Declarative account validation
- Automatic serialization
- Constraint macros (`has_one`, `seeds`, `owner`)
- Reduced boilerplate

Anchor makes development faster and safer **when used correctly**.

However:
- It can hide important security reasoning behind macros
- Developers may rely on defaults without understanding what is enforced
- Incorrect constraints can still result in exploitable programs

Anchor reduces boilerplate  but it does not eliminate the need to understand security.

https://www.helius.dev/blog/an-introduction-to-anchor-a-beginners-guide-to-building-solana-programs

Helius Blog – “Introduction to Anchor: A Beginner’s Guide” is an in-depth, long-form article by 0xIchigo (January 2024) that serves as a
strong entry point for developers who already understand core Solana concepts and want to learn what Anchor brings to the table.

---

### Pinocchio

Pinocchio is a lower-level framework focused on explicitness.

It:
- Requires manual account validation
- Forces developers to reason about ownership and authority
- Has minimal abstraction over Solana primitives

This makes Pinocchio:
- More verbose
- Easier to audit line-by-line
- Less forgiving of mistakes





---
## A brief Run through of the examples 
<div style="position: relative; padding-bottom: 56.25%; height: 0;"><iframe src="https://www.loom.com/embed/e9b5298bfebc456e8b71fb342298f8ea" frameborder="0" webkitallowfullscreen mozallowfullscreen allowfullscreen style="position: absolute; top: 0; left: 0; width: 100%; height: 100%;"></iframe></div>


## Repository Structure

Each vulnerability is isolated and easy to explore:


Attack Example/
├── README.md                  # Explanation of the attack
├── Programs/
│   ├── MainExample/
│   │   ├── src/
│   │   │   └── lib.rs         # Vulnerable implementation
│   │   └── Cargo.toml
│   ├── MainExample-Fixed/
│   │   ├── src/
│   │   │   └── lib.rs         # Fixed implementation
│   │   ├── tests/
│   │   │   └── main_example_fixed.ts  # Automated tests for fixed version
│   │   └── Cargo.toml
└── README.md                   

```

### Notes

- **Attack Example/README.md** → Explains the vulnerability (e.g., Confused Deputy, CPI issues).
- **Programs/MainExample/src/lib.rs** → Original vulnerable smart contract code.
- **Programs/MainExample-Fixed/src/lib.rs** → Secure/fixed version of the contract.
- **Programs/MainExample-Fixed/tests/** → Automated test scripts validating fixes.

---

