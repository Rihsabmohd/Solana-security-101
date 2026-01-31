

# Confused Deputy Attack via Malicious CPI 

This repository demonstrates a **Confused Deputy vulnerability** on Solana caused by **untrusted Cross-Program Invocation (CPI)** — and shows a **secure, fixed implementation** using explicit program validation.


---

## Background: Confused Deputy on Solana

A **Confused Deputy** attack occurs when a privileged program is tricked into performing actions on behalf of an untrusted actor, using authority that should never have been delegated.

On Solana, this commonly appears when:
- A program performs CPI
- The target `program_id` is user-controlled
- Signer privileges are forwarded
- Instruction behavior is *assumed* rather than enforced

---

## High-Level Architecture

Three programs are involved:

1. **Marketplace Program (Deputy)**
   - Manages NFT listings and escrow
   - Handles purchases
   - Performs CPI to distribute royalties

2. **Legitimate Royalty Program**
   - Transfers the *exact* royalty amount
   - Enforces basic authorization checks

3. **Fake (Malicious) Royalty Program**
   - Implements the same instruction interface
   - Exploits signer privileges
   - Transfers more lamports than intended

---

## Vulnerable Marketplace Logic

The marketplace allows users to specify a `royalty_program` account and blindly invokes it during purchase.

### Vulnerable CPI Pattern

```rust
let royalty_instruction = Instruction {
    program_id: ctx.accounts.royalty_program.key(),
    accounts: vec![
        AccountMeta::new(ctx.accounts.buyer.key(), true),
        AccountMeta::new(ctx.accounts.escrow.key(), false),
        AccountMeta::new_readonly(ctx.accounts.seller.key(), false),
        AccountMeta::new_readonly(listing.key(), false),
        AccountMeta::new_readonly(ctx.accounts.system_program.key(), false),
    ],
    data: instruction_data,
};

anchor_lang::solana_program::program::invoke(
    &royalty_instruction,
    &[
        ctx.accounts.buyer.to_account_info(),
        ctx.accounts.escrow.to_account_info(),
        ctx.accounts.seller.to_account_info(),
        ctx.accounts.listing.to_account_info(),
    ],
)?;
````

### Why This Is Unsafe

* `royalty_program` is **user-supplied**
* Buyer is forwarded as a **signer**
* Marketplace assumes royalty logic is benign
* No validation of CPI target program

This turns the marketplace into a **confused deputy**.

---

## Legitimate Royalty Program (Expected Behavior)

The legitimate royalty program transfers **only the specified royalty amount** and performs minimal checks.

```rust
let transfer_instruction =
    system_instruction::transfer(buyer.key, escrow.key, royalty_amount);

anchor_lang::solana_program::program::invoke(
    &transfer_instruction,
    &[
        buyer.clone(),
        escrow.clone(),
        system_program.to_account_info(),
    ],
)?;
```

This is the behavior the marketplace expects — but **does not enforce**.

---

## Fake Royalty Program (Malicious CPI Target)

The fake royalty program:

* Uses the same instruction name (`distribute_royalties`)
* Accepts the same account layout
* Exploits signer privileges to drain more funds

### Malicious Logic

```rust
let actual_amount_to_take = royalty_amount * 3;

let transfer_instruction =
    system_instruction::transfer(buyer.key, escrow.key, amount_to_transfer);

anchor_lang::solana_program::program::invoke(
    &transfer_instruction,
    &[
        buyer.clone(),
        escrow.clone(),
        system_program.to_account_info(),
    ],
)?;
```

### Why the Attack Works

* Marketplace passes buyer as signer
* No upper bound enforced on transferred amount
* CPI success is treated as correctness
* Funds are drained without violating runtime rules

---

## Fixed Version: Securing the CPI Boundary

The fix enforces **explicit trust** at the CPI boundary by validating the royalty program before invocation.

### Trusted Program Whitelisting

```rust
const TRUSTED_ROYALTY_PROGRAM: &str =
    "8GQBYWVbgsZvem5Vef4SiL5YmD1pgAkxMnB5NBydF5HQ";
```

### Enforced Constraint in Accounts Context

```rust
#[account(
    constraint = royalty_program.key().to_string() == TRUSTED_ROYALTY_PROGRAM
        @ MarketplaceError::UntrustedRoyaltyProgram
)]
pub royalty_program: AccountInfo<'info>;
```

This ensures:

* Only the legitimate royalty program can ever be invoked
* User-supplied malicious programs are rejected at runtime
* CPI signer delegation becomes safe again

---

## Secure CPI Invocation

```rust
let royalty_instruction = Instruction {
    program_id: ctx.accounts.royalty_program.key(),
    accounts: vec![
        AccountMeta::new(ctx.accounts.buyer.key(), true),
        AccountMeta::new(ctx.accounts.escrow.key(), false),
        AccountMeta::new_readonly(ctx.accounts.seller.key(), false),
        AccountMeta::new_readonly(listing.key(), false),
        AccountMeta::new_readonly(ctx.accounts.system_program.key(), false),
    ],
    data: instruction_data,
};

anchor_lang::solana_program::program::invoke(
    &royalty_instruction,
    &[
        ctx.accounts.buyer.to_account_info(),
        ctx.accounts.escrow.to_account_info(),
        ctx.accounts.seller.to_account_info(),
        ctx.accounts.listing.to_account_info(),
    ],
)?;
```

Because the program ID is **validated**, signer delegation is no longer exploitable.

---

## Root Cause Summary

The vulnerability existed because the marketplace:

* Trusted user-controlled program IDs
* Delegated signer authority without validation


The fix addresses this by:

*  Explicitly validating the CPI target and 
*  Enforcing program identity


---

## Key Security Takeaways

* CPI boundaries are **security boundaries**
* Anchor does **not** secure CPI targets automatically
* Signer privileges must never be delegated blindly
* Program ID validation is mandatory for sensitive CPIs

---

