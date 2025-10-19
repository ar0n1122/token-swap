# Solana Token Swap Program - Complete Educational Guide

## Table of Contents

1. [Program Overview](#program-overview)
2. [Architecture & File Structure](#architecture--file-structure)
3. [Core Concepts You Need to Understand](#core-concepts-you-need-to-understand)
4. [Program Flow & Logic](#program-flow--logic)
5. [Detailed Code Analysis](#detailed-code-analysis)
6. [Step-by-Step Program Breakdown](#step-by-step-program-breakdown)
7. [Test Flow Analysis](#test-flow-analysis)
8. [Learning Resources](#learning-resources)

---

## Program Overview

This is a **token swap program** built using Anchor framework on Solana. It implements a simple **escrow-based token exchange system** where:

- **Alice** (Maker) wants to trade Token A for Token B
- **Bob** (Taker) wants to trade Token B for Token A
- The program acts as a **trusted intermediary** using PDAs (Program Derived Addresses)

### Key Features:

- ✅ Escrow-based swaps (tokens are held securely)
- ✅ Atomic transactions (all-or-nothing)
- ✅ PDA-based vault system
- ✅ Support for Token Extensions Program (Token 2022)
- ✅ Automatic cleanup after completion

---

## Architecture & File Structure

```
programs/swap/src/
├── lib.rs              # Main program entry point
├── constants.rs        # Program constants
├── error.rs           # Custom error definitions
├── state/
│   └── offer.rs       # Offer account state structure
└── instructions/
    ├── mod.rs         # Module exports
    ├── make_offer.rs  # Logic for creating offers
    ├── take_offer.rs  # Logic for accepting offers
    └── shared.rs      # Shared utility functions
```

### Program Flow:

```
Alice (Maker) → make_offer() → Creates Offer + Vault → Bob (Taker) → take_offer() → Swap Complete
```

---

## Core Concepts You Need to Understand

### 1. **Program Derived Addresses (PDAs)**

📖 **Resource**: [Solana Cookbook - PDAs](https://solanacookbook.com/core-concepts/pdas.html)

**What it is**: Deterministic addresses that programs can "own" and sign for
**Why important**: Enables programs to hold and transfer tokens without private keys

```rust
// PDA for the offer account
seeds = [b"offer", maker.key().as_ref(), id.to_le_bytes().as_ref()]
```

### 2. **Cross-Program Invocation (CPI)**

📖 **Resource**: [Anchor Book - CPI](https://www.anchor-lang.com/docs/cross-program-invocations)

**What it is**: One program calling another program's instruction
**In this program**: Calling Token Program to transfer tokens

```rust
let cpi_context = CpiContext::new_with_signer(
    context.accounts.token_program.to_account_info(),
    accounts,
    &signer_seeds, // PDA signs the transaction
);
```

### 3. **Associated Token Accounts (ATAs)**

📖 **Resource**: [SPL Token Guide](https://spl.solana.com/associated-token-account)

**What it is**: Standardized way to derive token account addresses
**Formula**: `ATA = f(owner_pubkey, mint_pubkey)`

### 4. **Token Program vs Token Extensions Program**

📖 **Resource**: [Token Extensions Overview](https://solana.com/developers/guides/token-extensions/getting-started)

**Token Program**: Original SPL Token program
**Token Extensions**: New enhanced version (Token 2022) with additional features

### 5. **Anchor Account Constraints**

📖 **Resource**: [Anchor Book - Account Constraints](https://www.anchor-lang.com/docs/account-constraints)

**What it is**: Compile-time checks that ensure account validity
**Examples**:

- `#[account(mut)]` - Account must be mutable
- `#[account(signer)]` - Account must sign transaction
- `#[account(has_one = maker)]` - Cross-account validation

### 6. **Rent and Account Closure**

📖 **Resource**: [Solana Docs - Rent](https://docs.solana.com/implemented-proposals/rent)

**What it is**: Solana charges rent for account storage
**Rent Exemption**: Accounts with 2+ years of rent are exempt
**Account Closure**: Returning rent to specified account

---

## Program Flow & Logic

### Phase 1: Making an Offer (Alice's Perspective)

```mermaid
graph TD
    A[Alice wants to swap Token A for Token B] --> B[Calls make_offer()]
    B --> C[Creates Offer PDA account]
    B --> D[Creates Vault ATA owned by Offer PDA]
    B --> E[Transfers Token A from Alice to Vault]
    B --> F[Stores offer details in Offer account]
    F --> G[Offer is now live and discoverable]
```

### Phase 2: Taking an Offer (Bob's Perspective)

```mermaid
graph TD
    A[Bob finds Alice's offer] --> B[Calls take_offer()]
    B --> C[Transfers Token B from Bob to Alice]
    B --> D[Offer PDA signs to transfer Token A from Vault to Bob]
    B --> E[Closes Vault account - rent goes to Bob]
    B --> F[Closes Offer account - rent goes to Alice]
    F --> G[Swap completed successfully]
```

---

## Detailed Code Analysis

### 1. Main Program Entry Point (`lib.rs`)

```rust
#[program]
pub mod swap {
    use super::*;

    pub fn make_offer(
        mut context: Context<MakeOffer>,
        id: u64,
        token_a_offered_amount: u64,
        token_b_wanted_amount: u64,
    ) -> Result<()> {
        instructions::make_offer::send_offered_tokens_to_vault(&context, token_a_offered_amount)?;
        instructions::make_offer::save_offer(&mut context, id, token_b_wanted_amount)?;
        Ok(())
    }

    pub fn take_offer(context: Context<TakeOffer>) -> Result<()> {
        instructions::take_offer::send_wanted_tokens_to_maker(&context)?;
        instructions::take_offer::withdraw_and_close_vault(&context)?;
        Ok(())
    }
}
```

**Key Points**:

- Two main instructions: `make_offer` and `take_offer`
- Clean separation of concerns with helper functions
- Error propagation using `?` operator

### 2. Offer State Structure (`state/offer.rs`)

```rust
#[account]
#[derive(InitSpace)]
pub struct Offer {
    pub id: u64,                    // Unique identifier
    pub maker: Pubkey,              // Who made the offer
    pub token_mint_a: Pubkey,       // Token being offered
    pub token_mint_b: Pubkey,       // Token wanted in return
    pub token_b_wanted_amount: u64, // How much of token B wanted
    pub bump: u8,                   // PDA bump for signing
}
```

**Key Points**:

- `#[account]` makes this an Anchor account type
- `#[derive(InitSpace)]` automatically calculates space needed
- Stores all necessary data for the swap

#### 🧠 **Design Decision: Why No `token_a_offered_amount` Field?**

You might notice that the `Offer` struct stores `token_b_wanted_amount` but **doesn't store** `token_a_offered_amount`. This is an intentional and efficient design choice:

**Why `token_a_offered_amount` is not stored:**

1. **Physical Token Deposit**: When `make_offer()` is called, the entire `token_a_offered_amount` is immediately transferred to the vault (escrow). The tokens are physically held there.

2. **Vault Balance = Offered Amount**: The vault's token account balance inherently represents the offered amount. No need for redundant storage.

3. **Atomic Withdrawal**: In `take_offer()`, the code transfers the entire vault balance (`vault.amount`) to the taker:
   ```rust
   transfer_checked(cpi_context, context.accounts.vault.amount, ...)
   ```

**Why `token_b_wanted_amount` IS stored:**

1. **No Physical Deposit**: Token B is never deposited in escrow initially
2. **Taker Needs Information**: The taker must know exactly how much Token B to provide
3. **Transfer Reference**: Used in `send_wanted_tokens_to_maker()` to transfer the correct amount

**Benefits of this Design:**

- ✅ **Storage Efficient**: Saves 8 bytes per offer (no redundant amount storage)
- ✅ **Prevents Discrepancies**: Can't have mismatches between stored vs. actual vault balance
- ✅ **Atomic Operations**: Vault balance always reflects true available amount
- ✅ **Gas Optimized**: Fewer storage operations = lower transaction costs

**Potential Considerations:**

- If you needed partial fills or complex swap logic, you might want to store the original offered amount
- For audit trails, storing both amounts could be helpful
- The current design assumes the entire vault balance is always the offered amount

This pattern demonstrates efficient state management in Solana programs where physical token custody eliminates the need for separate accounting.

### 3. Make Offer Logic (`instructions/make_offer.rs`)

#### Account Validation:

```rust
#[derive(Accounts)]
#[instruction(id: u64)]
pub struct MakeOffer<'info> {
    #[account(mut)]
    pub maker: Signer<'info>,

    #[account(mint::token_program=token_program)]
    pub token_mint_a: InterfaceAccount<'info, Mint>,

    #[account(
        init,
        payer=maker,
        space= ANCHOR_DISCRIMINATOR + Offer::INIT_SPACE,
        seeds=[b"offer", maker.key().as_ref(), id.to_le_bytes().as_ref()],
        bump,
    )]
    pub offer: Account<'info, Offer>,

    #[account(
        init,
        payer=maker,
        associated_token::mint=token_mint_a,
        associated_token::authority=offer,
        associated_token::token_program=token_program
    )]
    pub vault: InterfaceAccount<'info, TokenAccount>,
    // ... other accounts
}
```

**Key Validations**:

- `maker` must sign the transaction
- `offer` account is created with PDA derived from maker + id
- `vault` is an ATA owned by the offer PDA (not the maker!)
- Space calculation includes discriminator (8 bytes) + actual data

#### 🔍 **Deep Dive: `#[account(mint::token_program=token_program)]` Constraint**

This constraint is crucial for **Token Extensions Program (Token 2022)** compatibility and security:

```rust
#[account(mint::token_program=token_program)]
pub token_mint_a: InterfaceAccount<'info, Mint>,
```

**What it does:**

1. **Program Association**: Validates that the mint account (`token_mint_a`) is owned by the specified `token_program`
2. **Token Program Flexibility**: Allows the program to work with both:
   - Legacy SPL Token Program (`TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA`)
   - Token Extensions Program/Token 2022 (`TokenzQdBNbLqP5VEhdkAS6EPFLC1PHnBqCXEpPxuEb`)

**Why it's important:**

**Security**: Without this constraint, a malicious user could pass a mint from a different program, potentially:

- Bypassing token program security checks
- Using fake/malicious token implementations
- Breaking transfer logic that expects specific program behavior

**Flexibility**: The same program code works with both token programs:

```rust
// Works with either program
pub token_program: Interface<'info, TokenInterface>,

// The constraint ensures mint matches the program
#[account(mint::token_program=token_program)]
pub token_mint_a: InterfaceAccount<'info, Mint>,
```

**Real-world scenario:**

```typescript
// Client can choose which token program to use
const tokenProgram = TOKEN_PROGRAM_ID; // Legacy
// OR
const tokenProgram = TOKEN_2022_PROGRAM_ID; // Extensions

// Pass the chosen program to the instruction
await program.methods.makeOffer(id, amountA, amountB).accounts({
  tokenProgram, // Program choice
  tokenMintA: mintA, // Must be owned by tokenProgram
  // ...
});
```

**Without this constraint:**

- ❌ Mint from Token Program + Token 2022 accounts = Runtime error
- ❌ Malicious mint account = Potential exploit
- ❌ Inconsistent program usage = Unpredictable behavior

**With this constraint:**

- ✅ Anchor validates mint ownership at instruction validation time
- ✅ Consistent program usage across all token operations
- ✅ Future-proof for new token program versions
- ✅ Clear error messages for mismatched accounts

This pattern is essential in modern Solana programs that need to support both token program versions while maintaining security and consistency.

#### Token Transfer Logic:

```rust
pub fn send_offered_tokens_to_vault(
    ctx: &Context<MakeOffer>,
    token_a_offered_amount: u64,
) -> Result<()> {
    transfer_tokens(
        &ctx.accounts.maker_token_account_a,    // From: Alice's account
        &ctx.accounts.vault,                    // To: Escrow vault
        &token_a_offered_amount,
        &ctx.accounts.token_mint_a,
        &ctx.accounts.maker,                    // Alice signs
        &ctx.accounts.token_program,
    )?;
    Ok(())
}
```

### 4. Take Offer Logic (`instructions/take_offer.rs`)

#### Account Validation:

```rust
#[derive(Accounts)]
pub struct TakeOffer<'info> {
    #[account(
        mut,
        close = maker,                          // Rent goes to maker
        has_one = maker,                        // Ensure maker matches
        has_one = token_mint_a,
        has_one = token_mint_b,
        seeds = [b"offer", maker.key().as_ref(), offer.id.to_le_bytes().as_ref()],
        bump = offer.bump
    )]
    offer: Account<'info, Offer>,
    // ... other accounts
}
```

**Key Validations**:

- `close = maker` - When offer account closes, rent goes to original maker
- `has_one` constraints ensure offer data integrity
- Seeds verification ensures we're using the correct offer PDA

#### Vault Withdrawal with PDA Signing:

```rust
pub fn withdraw_and_close_vault(context: &Context<TakeOffer>) -> Result<()> {
    let seeds = &[
        b"offer",
        context.accounts.maker.to_account_info().key.as_ref(),
        &context.accounts.offer.id.to_le_bytes()[..],
        &[context.accounts.offer.bump],         // Stored bump
    ];
    let signer_seeds = [&seeds[..]];

    // Transfer tokens from vault to taker using PDA signature
    let accounts = TransferChecked {
        from: context.accounts.vault.to_account_info(),
        to: context.accounts.taker_token_account_a.to_account_info(),
        mint: context.accounts.token_mint_a.to_account_info(),
        authority: context.accounts.offer.to_account_info(), // PDA is authority
    };

    let cpi_context = CpiContext::new_with_signer(
        context.accounts.token_program.to_account_info(),
        accounts,
        &signer_seeds,    // PDA signs on behalf of the program
    );

    transfer_checked(cpi_context, vault_amount, decimals)?;

    // Close the vault account
    close_account(cpi_context)?;
}
```

**Critical Concept**: The offer PDA "signs" the transaction to authorize the token transfer from the vault.

#### `withdraw_and_close_vault` Step-by-Step

**Step 1: Create Seeds Array**

```rust
let seeds = &[
    b"offer",
    context.accounts.maker.to_account_info().key.as_ref(),
    &context.accounts.offer.id.to_le_bytes()[..],
    &[context.accounts.offer.bump],
];
```

- `b"offer"`: Static string literal as first seed (discriminator)
- `maker.key.as_ref()`: The maker's public key as bytes (32 bytes)
- `offer.id.to_le_bytes()`: The offer ID converted to little-endian bytes (8 bytes)
- `[offer.bump]`: The bump seed that makes this a valid PDA (1 byte)

These seeds uniquely identify this specific offer PDA.

**Step 2: Create Signer Seeds**

```rust
let signer_seeds = [&seeds[..]];
```

Wraps the seeds in the format required for `CpiContext::new_with_signer()`.

**Step 3: Setup Transfer Accounts**

```rust
let accounts = TransferChecked {
    from: context.accounts.vault.to_account_info(),           // Source: vault holding tokens
    to: context.accounts.taker_token_account_a.to_account_info(), // Dest: taker's account
    mint: context.accounts.token_mint_a.to_account_info(),    // Token type verification
    authority: context.accounts.offer.to_account_info(),      // Who authorizes: offer PDA
};
```

**Step 4: Create CPI Context with Signer**

```rust
let cpi_context = CpiContext::new_with_signer(
    context.accounts.token_program.to_account_info(),
    accounts,
    &signer_seeds,
);
```

**CPI Signing:**

- The offer PDA is the signing authority
- The program signs on behalf of the offer PDA
- Program provides seeds to prove it can act as the PDA
- Solana runtime verifies: `hash(seeds + program_id) = PDA address`

**Step 5: Execute Transfer**

```rust
transfer_checked(
    cpi_context,
    context.accounts.vault.amount,      // Transfer ALL tokens
    context.accounts.token_mint_a.decimals, // For validation
)?;
```

Transfers all tokens from vault to taker's account.

**Step 6: Close Account**

```rust
let accounts = CloseAccount {
    account: context.accounts.vault.to_account_info(),     // Account to close
    destination: context.accounts.taker.to_account_info(), // Rent goes here
    authority: context.accounts.offer.to_account_info(),   // PDA authority
};

let cpi_context = CpiContext::new_with_signer(
    context.accounts.token_program.to_account_info(),
    accounts,
    &signer_seeds, // Same PDA signing
);

close_account(cpi_context)
```

**Complete Flow Summary:**

1. **Derive PDA signing authority** using the original seeds
2. **Transfer all escrowed tokens** from vault → taker's account
3. **Close empty vault account** and send rent SOL → taker
4. **Clean up** - no leftover accounts or state

**Key Security Features:**

- Only the program can sign for the PDA (using correct seeds)
- PDA cannot be controlled by external actors
- Atomic execution - either everything succeeds or everything fails
- Automatic cleanup prevents state bloat

### 5. Shared Utilities (`instructions/shared.rs`)

```rust
pub fn transfer_tokens<'info>(
    from: &InterfaceAccount<'info, TokenAccount>,
    to: &InterfaceAccount<'info, TokenAccount>,
    amount: &u64,
    mint: &InterfaceAccount<'info, Mint>,
    authority: &Signer<'info>,
    token_program: &Interface<'info, TokenInterface>,
) -> Result<()> {
    let cpi_context = CpiContext::new(
        token_program.to_account_info(),
        TransferChecked {
            from: from.to_account_info(),
            to: to.to_account_info(),
            mint: mint.to_account_info(),
            authority: authority.to_account_info(),
        }
    );
    transfer_checked(cpi_context, *amount, mint.decimals)?;
    Ok(())
}
```

**Why `transfer_checked`?**

- Includes decimal validation
- More secure than basic `transfer`
- Prevents common token transfer errors

---

## Step-by-Step Program Breakdown

### Step 1: Alice Creates an Offer

1. **Input Parameters**:

   - `id`: Unique offer identifier (u64)
   - `token_a_offered_amount`: How much of Token A Alice offers
   - `token_b_wanted_amount`: How much of Token B Alice wants

2. **Account Creation**:

   ```rust
   // Offer PDA: Deterministic address
   offer_pda = find_pda([b"offer", alice_pubkey, id_bytes])

   // Vault ATA: Token account owned by offer PDA
   vault_ata = find_ata(token_mint_a, offer_pda)
   ```

3. **Token Transfer**:

   ```rust
   // Alice → Vault
   transfer_tokens(alice_account, vault, amount, mint, alice, token_program)
   ```

4. **State Storage**:
   ```rust
   offer_account.set_inner(Offer {
       id,
       maker: alice_pubkey,
       token_mint_a,
       token_mint_b,
       token_b_wanted_amount,
       bump,
   });
   ```

### Step 2: Bob Discovers and Takes the Offer

1. **Account Resolution**:

   - Bob finds Alice's offer (off-chain discovery)
   - Derives the same PDA addresses Alice used
   - Prepares his token accounts

2. **Validation Checks**:

   ```rust
   // Anchor automatically validates:
   assert!(offer.maker == alice_pubkey);
   assert!(offer.token_mint_a == expected_mint_a);
   assert!(offer.token_mint_b == expected_mint_b);
   ```

3. **Token Exchanges**:

   ```rust
   // Step 1: Bob → Alice (Token B)
   transfer_tokens(bob_account_b, alice_account_b, wanted_amount, ...)

   // Step 2: Vault → Bob (Token A) - PDA signs!
   transfer_checked_with_signer(vault, bob_account_a, offered_amount, ...)
   ```

4. **Cleanup**:
   ```rust
   close_account(vault);  // Rent → Bob (taker)
   close_offer();         // Rent → Alice (maker)
   ```

---

## Test Flow Analysis

### Setup Phase (`tests/swap.ts`)

```typescript
// Create users with different token balances
Alice: 1,000,000,000 Token A, 0 Token B
Bob:   0 Token A, 1,000,000,000 Token B

// Define swap amounts
tokenAOfferedAmount = 1,000,000
tokenBWantedAmount = 1,000,000
```

### Test 1: Make Offer

```typescript
const transactionSignature = await program.methods
  .makeOffer(offerId, tokenAOfferedAmount, tokenBWantedAmount)
  .accounts({ ...accounts })
  .signers([alice])
  .rpc();
```

**Verification**:

- Vault contains Alice's offered tokens
- Offer account stores correct swap parameters

### Test 2: Take Offer

```typescript
const transactionSignature = await program.methods
  .takeOffer()
  .accounts({ ...accounts })
  .signers([bob])
  .rpc();
```

**Verification**:

- Bob receives Alice's Token A
- Alice receives Bob's Token B
- All accounts are properly closed

---

## Learning Resources

### Essential Solana Concepts

1. **Solana Account Model**: [Solana Docs](https://docs.solana.com/developing/programming-model/accounts)
2. **Program Derived Addresses**: [Solana Cookbook](https://solanacookbook.com/core-concepts/pdas.html)
3. **Token Program**: [SPL Token Guide](https://spl.solana.com/token)

### Anchor Framework

1. **Anchor Book**: [Official Documentation](https://www.anchor-lang.com/)
2. **Account Constraints**: [Anchor Constraints](https://www.anchor-lang.com/docs/account-constraints)
3. **CPI Deep Dive**: [Cross Program Invocations](https://www.anchor-lang.com/docs/cross-program-invocations)

### Advanced Topics

1. **Token Extensions**: [Token 2022 Guide](https://solana.com/developers/guides/token-extensions/getting-started)
2. **Security Best Practices**: [Neodyme Security Guide](https://neodyme.io/en/blog/solana_common_pitfalls/)
3. **Escrow Patterns**: [Paul X Escrow Tutorial](https://paulx.dev/blog/2021/01/14/programming-on-solana-an-introduction/#escrow-program)

### Development Tools

1. **Solana CLI**: [Installation Guide](https://docs.solana.com/cli/install-solana-cli-tools)
2. **Anchor CLI**: [Setup Instructions](https://www.anchor-lang.com/docs/installation)
3. **Testing Framework**: [@solana-developers/helpers](https://github.com/solana-developers/helpers)

---

## Key Security Considerations

### 1. **PDA Ownership**

- Vault is owned by offer PDA, not the maker
- Only the program can authorize transfers from vault
- Prevents maker from withdrawing after offer is made

### 2. **Account Validation**

- `has_one` constraints prevent tampering
- Seed verification ensures correct PDA usage
- Token mint validation prevents wrong token swaps

### 3. **Atomic Operations**

- Either both transfers succeed or entire transaction fails
- No partial swaps possible
- Built-in rollback mechanism

### 4. **Rent Optimization**

- Vault rent goes to taker (incentivizes completion)
- Offer rent goes back to maker (fair cost distribution)
- Prevents rent griefing attacks

This program demonstrates fundamental Solana/Anchor patterns that you'll see in many DeFi applications. Understanding this escrow pattern is crucial for building more complex protocols like AMMs, lending platforms, and NFT marketplaces.
