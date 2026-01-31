
# Duplicate Writable Account Vulnerability (Solana)

## Summary

Duplicate writable account vulnerabilities occur when the **same account is
passed multiple times as different writable accounts** within a single
instruction. If not handled carefully, this can lead to **state corruption,
double-counting, or bypassed invariants**.

This example demonstrates how transferring NFTs between vaults can break when
the source and destination vault are accidentally (or maliciously) the **same
account**, and how to fix it safely using Anchor’s **commit and reload pattern**.

---

## Overview

Solana allows the same account to appear multiple times in an instruction’s
account list — even under different names — as long as it satisfies the account
constraints.

When **both references are mutable**, Anchor deserializes them into **separate
Rust structs in memory**, even though they point to the same on-chain account.

Without care, this can cause:
- Lost updates
- Double writes
- Incorrect arithmetic
- Broken invariants

---

## Vulnerable Pattern: Duplicate Mutable Accounts

The vulnerable instruction allows transferring NFTs between two vaults:

```rust
#[derive(Accounts)]
pub struct TransferNftBetweenVaults<'info> {
    #[account(mut)]
    pub source_vault: Account<'info, NftVault>,
    #[account(mut)]
    pub destination_vault: Account<'info, NftVault>,
    pub owner: Signer<'info>,
}
````

There is **no check** preventing:

```text
source_vault.key() == destination_vault.key()
```

---

## Why This Is Dangerous

Anchor deserializes both accounts independently:

```text
source_vault  -> mutable copy #1
destination_vault -> mutable copy #2
```

If both point to the **same account**, each mutation overwrites the other
depending on write order.

---

## How the Attack Works

### Step 1: Attacker Passes Same Vault Twice

```text
source_vault = Vault A
destination_vault = Vault A
```

Both are marked `mut`.

---

### Step 2: Program Mutates Source

```rust
source.nft_count -= 1;
```

This update is only applied to **source’s in-memory copy**.

---

### Step 3: Program Mutates Destination

```rust
destination.nft_count += 1;
```

This operates on a **stale copy**, unaware of the earlier subtraction.

---

### Result: Broken Accounting

Instead of net zero change, the vault may:

* Gain NFTs
* Lose NFTs
* End with incorrect counts

This breaks the invariant:

> “Transfers should not mint or burn NFTs.”

---

## Why Anchor Does Not Automatically Protect You

Anchor **does not forbid** duplicate accounts by default.

It assumes developers will:

* Validate account relationships
* Handle aliasing safely
* Reload accounts when needed

This is a **logic-layer vulnerability**, not a framework bug.

---

## Fixed Pattern: Commit & Reload

The fixed version applies a **commit/reload pattern** to ensure correctness even
if the same account is passed twice.

---

## Fixed Transfer Logic

```rust
pub fn transfer_nft_between_vaults(
    ctx: Context<TransferNftBetweenVaults>,
    nft_mint: Pubkey,
) -> Result<()> {
    msg!("SECURE NFT transfer with commit/reload pattern");

    let source = &mut ctx.accounts.source_vault;
    let destination = &mut ctx.accounts.destination_vault;

    require!(source.nft_count > 0, ErrorCode::EmptyVault);
    require!(
        source.collection == destination.collection,
        ErrorCode::CollectionMismatch
    );

    // Step 1: Update source vault
    source.nft_count = source
        .nft_count
        .checked_sub(1)
        .ok_or(ErrorCode::VaultUnderflow)?;

    // FIXED: Commit source changes to storage
    source.exit(&ctx.program_id)?;

    // FIXED: Reload destination from storage
    ctx.accounts.destination_vault.reload()?;

    // Step 2: Update destination with fresh state
    let destination = &mut ctx.accounts.destination_vault;
    destination.nft_count = destination
        .nft_count
        .checked_add(1)
        .ok_or(ErrorCode::VaultOverflow)?;

    Ok(())
}
```

---

## Why This Fix Works

### Commit (`exit`)

```rust
source.exit(&ctx.program_id)?;
```

* Writes source changes immediately to account storage
* Prevents later writes from overwriting state

---

### Reload

```rust
ctx.accounts.destination_vault.reload()?;
```

* Forces Anchor to re-deserialize the account from storage
* Ensures the destination has **fresh, correct data**
* Handles the case where source == destination safely

---

## Alternative Mitigations

Depending on your use case, you can also:

* Explicitly prevent duplicates:

  ```rust
  require!(
      source.key() != destination.key(),
      ErrorCode::DuplicateVault
  );
  ```

* Use PDAs with deterministic separation

* Separate instructions for self-transfers

* Use read-only references where possible

---

## Key Takeaways

* Solana allows duplicate accounts in instructions
* Writable duplicates can corrupt state
* Anchor deserializes accounts independently
* Mutating aliased accounts without care is unsafe
* **Commit + reload guarantees correctness**
* Never assume accounts are distinct unless you check

If your instruction mutates more than one account, always consider:

> “What if they’re the same?”


