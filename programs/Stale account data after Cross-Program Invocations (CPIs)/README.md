
# Token-Based NFT Marketplace: CPI & Escrow Security (Solana / Anchor)

This repository demonstrates a **vulnerability in token-based NFT marketplaces** using Anchor on Solana, and shows a **fixed, secure implementation**.

The example focuses on **CPI-based token transfers** from buyers to escrow accounts and illustrates the importance of **post-CPI validation** to prevent **fund mismatches** or unintended token transfers.

---

## Background

On Solana, Cross-Program Invocations (CPI) are used to interact with SPL Token accounts.  
However, blindly trusting the results of a CPI **without verifying the token balances** can lead to:

- Users paying **more or less tokens** than intended
- Escrow accounts not receiving the correct amounts
- Logical inconsistencies that could be exploited in advanced scenarios

This is particularly relevant for **NFT marketplaces** where token payments must exactly match the listed price.

---

## High-Level Architecture

### Accounts

1. **NftListing**
   - Stores seller, seller token account, price, activity status, and PDA bump
2. **Buyer Token Account**
   - Tokens owned by the buyer, used for payments
3. **Escrow Token Account**
   - Token account that receives payments

### Flow

1. Seller creates a listing with a token price.
2. Buyer calls `purchase_nft` to pay tokens.
3. Marketplace performs a CPI to the SPL Token program to transfer tokens.
4. Marketplace updates the listing as inactive after purchase.

---

## Vulnerable Implementation

The original function (`purchase_nft`) relied on:

- **Trusting the SPL Token program CPI** to succeed
- **Not reloading the token account balances** after the transfer
- **Not verifying escrow actually received the correct amount**

### Vulnerable Pattern

```rust
// Make CPI to transfer tokens from buyer to escrow
let transfer_accounts = Transfer {
    from: ctx.accounts.buyer_token_account.to_account_info(),
    to: ctx.accounts.escrow_token_account.to_account_info(),
    authority: ctx.accounts.buyer.to_account_info(),
};

let cpi_ctx = CpiContext::new(ctx.accounts.token_program.to_account_info(), transfer_accounts);
token::transfer(cpi_ctx, listing.price)?;

// No reload here: balances may not reflect the real CPI outcome
let buyer_token_balance_after = ctx.accounts.buyer_token_account.amount;
let escrow_token_balance_after = ctx.accounts.escrow_token_account.amount;
````

### Why This Is Unsafe

* **CPI may not transfer the exact amount** due to slippage, errors, or token program anomalies.
* Marketplace assumes `listing.price` was correctly transferred.
* No post-CPI validation means the **buyer could overpay** or the escrow could be underfunded.
* This could become a **funds mismatch vulnerability**, especially in automated marketplace logic.

---

## Fixed Version: Post-CPI Balance Validation

The fix adds **reloads for token accounts after the CPI**, ensuring balances reflect the actual transfer.

```rust
ctx.accounts.buyer_token_account.reload()?;
ctx.accounts.escrow_token_account.reload()?;
```

Then, it calculates the transferred amounts:

```rust
let buyer_spent = buyer_token_balance_before - buyer_token_balance_after;
let escrow_received = escrow_token_balance_after - escrow_token_balance_before;

if escrow_received != listing.price {
    return Err(error!(MarketplaceError::TransferAmountMismatch));
}
```

### Why This Is Safe

* Forces **post-CPI verification** of token balances
* Prevents escrow underfunding or buyer overpayment
* Ensures **listing is marked inactive only after successful validation**
* Robust even if token CPI is manipulated or partially fails

---

## Key Security Takeaways

* **CPI is not atomic in logic**: Always verify token balances after the call.
* **Never assume SPL Token transfers succeed exactly as requested**.
* **Validate amounts received in escrow** before updating state.
* **Reload token accounts after CPI** for accurate state.

---

## Example Flow (Fixed)

1. Buyer wants to buy an NFT priced at 100 tokens.
2. Marketplace calls SPL Token `transfer` via CPI.
3. Reload buyer and escrow token accounts to reflect real balances.
4. Verify `escrow_received == listing.price`.
5. Mark listing inactive.
6. Transaction succeeds only if actual transfer matches expected transfer.




