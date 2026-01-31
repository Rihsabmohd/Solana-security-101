
# Missing Signer Validation 

## Summary

Missing signer validation is a critical authorization vulnerability in Solana
programs. It occurs when a program verifies that an account *matches* an expected
authority, but fails to require that authority to **cryptographically sign** the
transaction.

This example demonstrates how using `UncheckedAccount` for an authority allows
any attacker to change ownership of a profile without possessing the private key,
and how requiring a `Signer` fully prevents this attack.

---

## Overview

In Solana, **accounts do not imply authorization**.

An account being passed into an instruction only proves that:
- The account exists
- The public key matches

It does **not** prove that the owner of that key approved the transaction.

Only a `Signer` constraint enforces cryptographic consent.

---

## Vulnerable Pattern: Authority Without Signature

In the vulnerable version, the `update_profile_authority` instruction allows
changing the stored authority of a user profile.

### Vulnerable Account Context

```rust
#[derive(Accounts)]
pub struct UpdateProfileAuthority<'info> {
    #[account(
        mut,
        has_one = authority
    )]
    pub user_profile: Account<'info, UserProfile>,
    /// CHECK: New authority to set
    pub new_authority: UncheckedAccount<'info>,
    /// CHECK: Current authority account
    pub authority: UncheckedAccount<'info>,
}
````

Although `has_one = authority` ensures that the `authority` account matches the
stored public key, it **does not require a signature**.

---

## Why This Is Dangerous

The program verifies **identity**, not **consent**.

```text
has_one = authority
```

Only checks:

* `user_profile.authority == authority.key()`

It does **not** check:

* Who signed the transaction

This means anyone can pass the correct public key and impersonate the authority.

---

## How the Attack Works

### Step 1: Victim Creates Profile

The profile is initialized with:

```text
UserProfile {
  authority = VictimPubkey
}
```

---

### Step 2: Attacker Calls update_profile_authority

The attacker submits a transaction with:

* `user_profile`: Victim’s profile
* `authority`: Victim’s public key (NOT signing)
* `new_authority`: Attacker’s public key

Because `authority` is an `UncheckedAccount`, **no signature is required**.

---

### Step 3: Authority Is Hijacked

```rust
profile.authority = ctx.accounts.new_authority.key();
```

The authority is updated successfully.

Final state:

```text
UserProfile {
  authority = AttackerPubkey
}
```

The attacker now fully controls the profile.

---

## Why has_one Is Not Enough

```rust
#[account(has_one = authority)]
```

This enforces **data consistency**, not authorization.



## Fixed Pattern: Require Authority Signature

The fix is to require the authority to be a `Signer`.

### Fixed Account Context

```rust
#[derive(Accounts)]
pub struct UpdateProfileAuthority<'info> {
    #[account(
        mut,
        has_one = authority
    )]
    pub user_profile: Account<'info, UserProfile>,
    /// CHECK: New authority to set
    pub new_authority: UncheckedAccount<'info>,
    // FIXED: Now requires signature from current authority
    pub authority: Signer<'info>,
}
```

---

## Why This Fix Works

### Cryptographic Authorization

By requiring:

```rust
pub authority: Signer<'info>
```

Anchor enforces that:

* The transaction must be signed by the authority’s private key
* Impersonation is impossible
* Replay or spoofing attacks fail automatically

---

### Automatic Runtime Enforcement

If the authority does not sign:

* The transaction fails
* The instruction handler never executes
* State cannot be modified

No manual checks are needed.

---

## Secure Update Logic

```rust
pub fn update_profile_authority(ctx: Context<UpdateProfileAuthority>) -> Result<()> {
    let profile = &mut ctx.accounts.user_profile;

    profile.authority = ctx.accounts.new_authority.key();
    Ok(())
}
```

This logic is now safe because authorization is enforced **before execution**.

---

## Key Takeaways

* Accounts ≠ authorization
* Public keys can be spoofed
* `UncheckedAccount` disables signer checks
* Always use Signer<'info> when requiring transaction signatures.
* **Every privileged authority must be a `Signer`**

