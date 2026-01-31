
# Account Re-Initialization Attacks 

## Summary

Account re-initialization is a common and dangerous vulnerability in Solana programs.
If a program allows an already-initialized account to be initialized again, an attacker
can overwrite existing state and silently take control of the account.

This example demonstrates how manual account initialization using `UncheckedAccount`
enables authority takeover, and how Anchor’s `init` constraint prevents this class of
attack entirely.

---

## Overview

On Solana, accounts are supplied by the user, not created implicitly by the program.
Programs must explicitly enforce:

- Whether an account is new or already initialized
- Who is allowed to initialize it
- Whether its data can safely be written

Initialization is a critical security boundary.  
If it can be executed more than once for the same account, all authorization guarantees
based on that account become unreliable.

---

## Vulnerable Pattern: Missing Initialization Guard

In the vulnerable version of this program, the `initialize_user_profile` instruction
manually initializes account data using an `UncheckedAccount`.

### Vulnerable Account Context

```rust
#[derive(Accounts)]
pub struct InitUserProfile<'info> {
    #[account(mut)]
    /// CHECK: User profile account
    pub user_profile: UncheckedAccount<'info>,
    pub authority: Signer<'info>,
    pub system_program: Program<'info, System>,
}
````

Using `UncheckedAccount` disables all of Anchor’s safety checks. Anchor does not verify
ownership, initialization state, or the account discriminator.

---

## Vulnerable Initialization Logic

```rust
let user_profile = &ctx.accounts.user_profile;
let mut data = user_profile.data.borrow_mut();

// Write account discriminator
let discriminator = <UserProfile as anchor_lang::Discriminator>::DISCRIMINATOR;
data[..8].copy_from_slice(&discriminator);

// Serialize and write profile data
profile.serialize(&mut cursor)?;
```

### Why This Is Dangerous

This logic blindly writes data into the account without checking whether:

* The account has already been initialized
* The account belongs to the expected program
* The discriminator already exists

Any signer can re-initialize an existing account and overwrite its state.

---

## How the Re-Initialization Attack Works

### Step 1: Legitimate Initialization

A legitimate user initializes their profile:

```text
UserProfile {
  authority = VictimPubkey
}
```

---

### Step 2: Attacker Re-Initializes the Same Account

The attacker calls `initialize_user_profile` again, passing:

* The same `user_profile` account
* Their own signer as `authority`

Because there is no guard preventing re-initialization, the program overwrites the
existing data.

---

### Step 3: Authority Takeover

After re-initialization, the account state becomes:

```text
UserProfile {
  authority = AttackerPubkey
}
```

The attacker now controls the account.

---

## Why Authorization Checks Fail

```rust
require!(
    profile.authority == ctx.accounts.authority.key(),
    MarketplaceError::Unauthorized
);
```

This check assumes the `authority` field was set once and never changed. Since the
account can be re-initialized, the attacker controls the stored authority, making this
authorization check ineffective.

---

## Secure Pattern: Anchor `init` Constraint

The fixed version uses Anchor’s `init` constraint to ensure the account is created and
initialized exactly once.

### Secure Account Context

```rust
#[derive(Accounts)]
pub struct InitUserProfile<'info> {
    #[account(
        init,
        payer = authority,
        space = 8 + 32 + (4 + 32) + 8 + 8 + 1,
        seeds = [b"profile", authority.key().as_ref()],
        bump
    )]
    pub user_profile: Account<'info, UserProfile>,

    #[account(mut)]
    pub authority: Signer<'info>,

    pub system_program: Program<'info, System>,
}
```

---

## Why This Fix Works

### One-Time Initialization Enforcement

The `init` constraint guarantees:

* The account does not already exist
* Initialization can only happen once
* Re-initialization attempts automatically fail


### Restoring Anchor Safety Guarantees

```rust
pub user_profile: Account<'info, UserProfile>
```

Using `Account<T>` restores  Ownership validation

---


## Key Takeaways

* Initialization is a security-critical operation
* Re-initialization enables silent authority takeover
* `UncheckedAccount` bypasses framework safety
* Manual data writes are high-risk
* Anchor’s `init` constraint prevents this vulnerability by default

If an account can be initialized twice, it can be stolen.



If you want the **next README** (authority checks, PDA collisions, CPI signer confusion, arithmetic overflow, etc.), just tell me which vulnerability you’re tackling next 🔐🚀
```
