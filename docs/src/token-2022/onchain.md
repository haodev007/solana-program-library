---
title: On-chain Program Guide
---

## Supporting Token and Token-2022 Together In Your Program

This guide is meant for on-chain program / dapp developers who want to support
Token and Token-2022 concurrently.

## Prerequisites

This guide requires the Solana CLI tool suite, minimum version 1.10.33 in order
to support all Token-2022 features.

## Motivation

On-chain program developers are accustomed to only including one token program,
to be used for all tokens in the application.

With the addition of Token-2022, developers must update on-chain programs. This
guide walks through the steps required to support both.

Important note: if you do not wish to support Token-2022, there is nothing to do.
Your existing on-chain program will loudly fail if an instruction includes any
Token-2022 mints / accounts.

Most likely, your program will fail with `ProgramError::IncorrectProgramId` while
trying to create a CPI instruction into the Token program, providing the Token-2022
program id.

## Structure of this Guide

To safely code the transition, we'll follow a test-driven development approach:

* add a dependency to `spl-token-2022`
* change tests to use `spl_token::id()` or `spl_token_2022::id()`, see that all
tests fail with Token-2022
* update on-chain program code to always use the instruction and deserializers from 
`spl_token_2022`, make all tests pass

Optionally, if an instruction uses more than one token mint, common to most DeFi,
you must add an input token program account for each additional mint. Since it's
possible to swap all types of tokens, we need to either invoke the correct token
program.

Everything here will reference real commits to the token-swap program, so
feel free to follow along and make the changes to your program.

## Part I: Support both token programs in single-token use cases

### Step 1: Update dependencies

In your `Cargo.toml`, add the latest `spl-token-2022` to your `dependencies`.
Check for the latest version of `spl-token-2022` in [crates.io](https://crates.io), since
that will typically be the version deployed to mainnet-beta.

### Step 2: Add test cases for Token and Token-2022

Using the `test-case` crate, you can update all tests to use both Token and
Token-2022. For example, a test defined as:

```
#[tokio::test]
async fn test_swap() {
    ...
}
```

Will become:

```
#[test_case(spl_token::id() ; "Token Program")]
#[test_case(spl_token_2022::id() ; "Token-2022 Program")]
#[tokio::test]
async fn test_swap(token_program_id: Pubkey) {
    ...
}
```

In your program-test setup, you must include `spl_token_2022.so` at the correct
address. You can add it as normal to `tests/fixtures/` after downloading it using:

```console
$ solana program dump TokenzQdBNbLqP5VEhdkAS6EPFLC1PHnBqCXEpPxuEb spl_token_2022.so
```

If you're using `solana-test-validator` for your tests, you can include it using:

```console
$ solana-test-validator -c TokenzQdBNbLqP5VEhdkAS6EPFLC1PHnBqCXEpPxuEb 
```

**Note**: This step is temporary, until Token-2022 is included by default in
`program-test` and `solana-test-validator`.

The token-swap does not use `program-test`, so there's a bit more
boilerplate, but the same principle applies.

### Step 3: Replace instruction creators

Everywhere in the code that uses `spl_token::instruction` must now use
`spl_token_2022::instruction`. The `"Token-2022 Program"` tests will still fail,
but importantly, the `"Token Program"` tests will pass using the new instruction
creators.

If your program uses unchecked transfers, you'll see a deprecation warning:

```
warning: use of deprecated function `spl_token_2022::instruction::transfer`: please use `transfer_checked` or `transfer_checked_with_fee` instead
```

If a token has a transfer fee, an unchecked transfer will fail. We'll fix that
later. If you want, in the meantime, feel free to add an `#[allow(deprecated)]`
to pass CI, with a TODO or issue to transition to `transfer_checked` everywhere.

### Step 4: Replace spl_token::id() with a parameter

Step 2 started the transition away from a fixed program id by adding
`token_program_id` as a parameter to the test function, but now you'll go
through your program and tests to use it everywhere.

Whenever `spl_token::id()` appears in the code, use a parameter corresponding
either to `spl_token::id()` or `spl_token_2022::id()`.

After this, all of your tests should pass! Not so fast though, there's one more
step needed to ensure compatibility.

### Step 5: Add Extensions to Tests

Although all of your tests are passing, you still need to account for differences
in accounts in token-2022.

Account extensions are stored after the first 165 bytes of the account, and the
normal `Account::unpack` and `Mint::unpack` will fail if the size of the account
is not exactly 165 and 82, respectively.

Let's make the tests fail again by adding an extension to all mint and token
accounts.  We'll add the `MintCloseAuthority` extension to mints, and the `ImmutableOwner`
extension to accounts.

When creating mint accounts, calculate the space required before allocating, then
include an `initialize_mint_close_authority` instruction before `initialize_mint`.
For example this could be:

```rust
use spl_token_2022::{extension::ExtensionType, instruction::*, state::Mint};
use solana_sdk::{system_instruction, transaction::Transaction};

// Calculate the space required using the `ExtensionType`
let space = ExtensionType::get_account_len::<Mint>(&[ExtensionType::MintCloseAuthority]);

// get the Rent object and calculate the rent required
let rent_required = rent.minimum_balance(space);

// and then create the account using those parameters
let create_instruction = system_instruction::create_account(&payer.pubkey(), mint_pubkey, rent_required, space, token_program_id);

// Important: you must initialize the mint close authority *BEFORE* initializing the mint,
// and only when working with Token-2022, since the instruction is unsupported by Token.
let initialize_close_authority_instruction = initialize_mint_close_authority(token_program_id, mint_pubkey, Some(close_authority)).unwrap();
let initialize_mint_instruction = initialize_mint(token_program_id, mint_pubkey, mint_authority_pubkey, freeze_authority, 9).unwrap();

// Make the transaction with all of these instructions
let create_mint_transaction = Transaction::new(&[create_instruction, initialize_close_authority_instruction, initialize_mint_instruction], Some(&payer.pubkey));

// Sign it and send it however you want!
```

The concept is similar with token accounts, but we'll use the `ImmutableOwner`
extension, which is actually supported by both programs, but `Tokenkeg...` will
no-op.

```rust
use spl_token_2022::{extension::ExtensionType, instruction::*, state::Account};
use solana_sdk::{system_instruction, transaction::Transaction};

// Calculate the space required using the `ExtensionType`
let space = ExtensionType::get_account_len::<Account>(&[ExtensionType::ImmutableOwner]);

// get the Rent object and calculate the rent required
let rent_required = rent.minimum_balance(space);

// and then create the account using those parameters
let create_instruction = system_instruction::create_account(&payer.pubkey(), account_pubkey, rent_required, space, token_program_id);

// Important: you must initialize immutable owner *BEFORE* initializing the account
let initialize_immutable_owner_instruction = initialize_immutable_owner(token_program_id, account_pubkey).unwrap();
let initialize_account_instruction = initialize_account(token_program_id, account_pubkey, mint_pubkey, owner_pubkey).unwrap();

// Make the transaction with all of these instructions
let create_account_transaction = Transaction::new(&[create_instruction, initialize_immutable_owner_instruction, initialize_account_instruction], Some(&payer.pubkey));

// Sign it and send it however you want!
```

After making these changes, everything fails again. Well done!

### Step 6: Use `StateWithExtensions` instead of `Mint` and `Account`

The test failures happen because the program is trying to deserialize a pure
`Mint` or `Account`, and failing because there are extensions added to it.

Token-2022 adds a new type called `StateWithExtensions`, which allows you to
deserialize the base type, and then pull out any extensions on the fly. It's
very close to the same cost as the normal `unpack`.

Everywhere in your code, wherever you see `Mint::unpack` or `Account::unpack`,
you'll have to change that to:

```rust
use spl_token_2022::{extension::StateWithExtensions, state::{Account, Mint}};
let account_state = StateWithExtensions::<Account>::unpack(&token_account_info.data.borrow())?;
let mint_state = StateWithExtensions::<Mint>::unpack(&mint_account_info.data.borrow())?;
```

Anytime you access fields in the state, you'll need to go through the `base`. For
example, to access the amount, you must do:

```rust
let token_amount = account_state.base.amount;
```

So typically, you'll just need to add in `.base` wherever those fields are accessed.

Once that's done, all of your tests should pass! Congratulations, your program
is now compatible with Token-2022!

If your program is using multiple token types at once, however, you will need to
do more work.

## Part II: Support Mixed Token Programs: trading a Token for a Token-2022

In Part I, we looked at the minimal amount of work to support Token-2022 in your
program. This work won't cover all cases, however. Specifically, in the token-swap
program, most instructions involve multiple token types. If those token types are
from different token programs, then our current implementation will fail.

For example, if you want to swap tokens from the Token program for tokens from
the Token-2022 program, then your program's instruction must provide each token
program, so that your program may invoke them.

Let's go through the steps to support both token programs in the same instruction.

### Step 1: Update all instruction interfaces

The first step is to update all instruction interfaces to accept a token program
for each token type used in the program.

For example, here is the previous definition for the `Swap` instruction:

```rust
///   Swap the tokens in the pool.
///
///   0. `[]` Token-swap
///   1. `[]` swap authority
///   2. `[]` user transfer authority
///   3. `[writable]` token_(A|B) SOURCE Account, amount is transferable by user transfer authority,
///   4. `[writable]` token_(A|B) Base Account to swap INTO.  Must be the SOURCE token.
///   5. `[writable]` token_(A|B) Base Account to swap FROM.  Must be the DESTINATION token.
///   6. `[writable]` token_(A|B) DESTINATION Account assigned to USER as the owner.
///   7. `[writable]` Pool token mint, to generate trading fees
///   8. `[writable]` Fee account, to receive trading fees
///   9. `[]` Token program id
///   10. `[optional, writable]` Host fee account to receive additional trading fees
Swap {
    pub amount_in: u64,
    pub minimum_amount_out: u64
}
```

`Swap` contains 3 different token types: token A, token B, and the pool token. Let's
add a separate token program for each, transforming the instruction into:

```rust
///   Swap the tokens in the pool.
///
///   0. `[]` Token-swap
///   1. `[]` swap authority
///   2. `[]` user transfer authority
///   3. `[writable]` token_(A|B) SOURCE Account, amount is transferable by user transfer authority,
///   4. `[writable]` token_(A|B) Base Account to swap INTO.  Must be the SOURCE token.
///   5. `[writable]` token_(A|B) Base Account to swap FROM.  Must be the DESTINATION token.
///   6. `[writable]` token_(A|B) DESTINATION Account assigned to USER as the owner.
///   7. `[writable]` Pool token mint, to generate trading fees
///   8. `[writable]` Fee account, to receive trading fees
///   9. `[]` Token (A|B) SOURCE program id
///   10. `[]` Token (A|B) DESTINATION program id
///   11. `[]` Pool Token program id
///   12. `[optional, writable]` Host fee account to receive additional trading fees
Swap {
    pub amount_in: u64,
    pub minimum_amount_out: u64
}
```

Note the clarification on `9.`, and the new inputs of `10.` and `11.`.

All of these additional accounts may make you wonder: how big will transactions
get with these new accounts? If you are using both Token and Token-2022,
the additional Token-2022 program will take up space in the transaction,
32 bytes for the pubkey, and 1 byte for its index.

On the flip side, if you're only using one token program at once, you will only
incur 1 byte of overhead because of the deduplication of accounts in the Solana
transaction format.

Also note that some instructions will remain unchanged. For example, here is the
`Initialize` instruction:

```rust
///   Initializes a new swap
///
///   0. `[writable, signer]` New Token-swap to create.
///   1. `[]` swap authority derived from `create_program_address(&[Token-swap account])`
///   2. `[]` token_a Account. Must be non zero, owned by swap authority.
///   3. `[]` token_b Account. Must be non zero, owned by swap authority.
///   4. `[writable]` Pool Token Mint. Must be empty, owned by swap authority.
///   5. `[]` Pool Token Account to deposit trading and withdraw fees.
///   Must be empty, not owned by swap authority
///   6. `[writable]` Pool Token Account to deposit the initial pool token
///   supply.  Must be empty, not owned by swap authority.
///   7. `[]` Token program id
Initialize { ... } // details omitted
```

Although we pass in token A and token B accounts, we don't actually need to invoke
their respective token programs. We do, however, mint new pool tokens, so we must
pass in the token program for the pool token mint.

This step is mostly churn since interfaces must be updated. Don't worry if some
tests fail after this step. We'll fix them in the next step.

### Step 2: Update instruction processors

If your instruction processor is expecting accounts after the added token programs,
you may see some test failures.

Specifically, in the token-swap example, the `Swap` instruction is expecting
an optional account at the end, which has been clobbered by the added token
programs.

For this step, we'll simply pull out all of the new provided accounts. For example,
in the `Swap` instruction processor, we'll go from:

```rust
let account_info_iter = &mut accounts.iter();
let swap_info = next_account_info(account_info_iter)?;
let authority_info = next_account_info(account_info_iter)?;
let user_transfer_authority_info = next_account_info(account_info_iter)?;                     let source_info = next_account_info(account_info_iter)?;
let swap_source_info = next_account_info(account_info_iter)?;
let swap_destination_info = next_account_info(account_info_iter)?;
let destination_info = next_account_info(account_info_iter)?;                                 let pool_mint_info = next_account_info(account_info_iter)?;                                   let pool_fee_account_info = next_account_info(account_info_iter)?;
let token_program_info = next_account_info(account_info_iter)?;                       
```

To:

```rust
let account_info_iter = &mut accounts.iter();
let swap_info = next_account_info(account_info_iter)?;
let authority_info = next_account_info(account_info_iter)?;
let user_transfer_authority_info = next_account_info(account_info_iter)?;
let source_info = next_account_info(account_info_iter)?;
let swap_source_info = next_account_info(account_info_iter)?;
let swap_destination_info = next_account_info(account_info_iter)?;
let destination_info = next_account_info(account_info_iter)?;
let pool_mint_info = next_account_info(account_info_iter)?;
let pool_fee_account_info = next_account_info(account_info_iter)?;
let source_token_program_info = next_account_info(account_info_iter)?;
let destination_token_program_info = next_account_info(account_info_iter)?;
let pool_token_program_info = next_account_info(account_info_iter)?;
```

For now, just use one of those. For example, we'll just use `pool_token_program_info`
everywhere. In the next step, we'll add some tests which will properly fail since
we're always using the same token program.

Once again, all of your tests should pass! But not for long.

### Step 3: Write tests using multiple token programs at once

In the spirit of test-driven development, let's start by writing some failing
tests.

Previously, our `test_case`s defined only provided one program id. Now it's
time to mix them up and add more cases. For full coverage, we could do all
permutations of different programs, but let's go with:

* all mints belong to Token
* all mints belong to Token-2022
* the pool mint belongs to Token, but token A and B belong to Token-2022
* the pool mint belongs to Token-2022, but token A and B are mixed

Let's update test cases to pass in three different program ids, and then use them
in the tests. For example, that means transforming:

```rust
#[test_case(spl_token::id(); "token")]
#[test_case(spl_token_2022::id(); "token-2022")]
fn test_initialize(token_program_id: Pubkey) {
```

Into:

```rust
#[test_case(spl_token::id(), spl_token::id(), spl_token::id(); "all-token")]
#[test_case(spl_token_2022::id(), spl_token_2022::id(), spl_token_2022::id(); "all-token-2022")]
#[test_case(spl_token::id(), spl_token_2022::id(), spl_token_2022::id(); "mixed-pool-token")]
#[test_case(spl_token_2022::id(), spl_token_2022::id(), spl_token::id(); "mixed-pool-token-2022")]
fn test_initialize(pool_token_program_id: Pubkey, token_a_program_id: Pubkey, token_b_program_id: Pubkey) {
    ...
}
```

This step may also involve churn, but take your time to go through it carefully,
and you'll have failing tests for the `mixed-pool-token` and `mixed-pool-token-2022`
test cases.

### Step 4: Use appropriate token program in your processor

Let's fix the failing tests! The errors come up because we're trying to operate
on tokens with the wrong program in a "mixed" Token and Token-2022 environment.

We need to properly use all of the `pool_token_program_info` / `token_a_program_info`
variables that we extracted in Step 2.

In the token-swap example, we'll check anywhere we filled in `pool_token_program_info`
by default, and instead choose the correct program info. For example, when
transferring the source tokens in `process_swap`, we currently have:

```rust
Self::token_transfer(
    swap_info.key,
    pool_token_program_info.clone(),
    source_info.clone(),
    swap_source_info.clone(),
    user_transfer_authority_info.clone(),
    token_swap.bump_seed(),
    to_u64(result.source_amount_swapped)?,
)?;
```

Let's use the correct token program, making this:

```rust
Self::token_transfer(
    swap_info.key,
    source_token_program_info.clone(),
    source_info.clone(),
    swap_source_info.clone(),
    user_transfer_authority_info.clone(),
    token_swap.bump_seed(),
    to_u64(result.source_amount_swapped)?,
)?;
```

While going through this, if you notice any owner checks for a token account or mint
in the form of:

```rust
if token_account_info.owner != &spl_token::id() { ... }
```

You'll need to update to a new owner check from `spl_token_2022`:

```rust
if spl_token_2022::check_spl_token_program_account(token_account_info.owner).is_err() { ... }
```

In this step, because of all the test cases in token-swap, we also have to update
the expected error due to mismatched owner token programs.

It's tedious, but at this point, we have updated our program to use both Token
and Token-2022 simultaneously. Congratulations! You're ready to be part of the
next stage of DeFi on Solana.

## Part III: Support Specific Extensions

### Update from `transfer` to `transfer_checked`

### Take fee into account when calculating slippage
