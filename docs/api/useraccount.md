---
id: user-account-init
title: Initialize User Account
sidebar_position: 6
---

# Initialize User Account

This instruction creates the `UserAccount` PDA for a trader.

It must be executed **before**:

- Depositing collateral
- Opening positions
- Placing limit orders
- Interacting with pool trading logic

> The full `UserAccount` struct layout and Borsh schema are defined in the **Schemas** section. This page focuses strictly on initialization.

---

# Instruction

## Discriminator

```
39
```

(From Rust enum ordering)

---

## Purpose

- Allocates the `UserAccount` PDA
- Sets:
  - `user`
  - `owner`
  - `token`
  - `created_at`
  - `bump`
- Initializes balances to zero
- Initializes vectors (orders, limit_orders, etc.)

---

## Required Accounts

| Index | Account | Signer | Writable |
|--------|----------|----------|-----------|
| 0 | user | ✅ | ✅ |
| 1 | user_account PDA | ❌ | ✅ |
| 2 | tokenMint | ❌ | ❌ |
| 3 | System Program | ❌ | ❌ |
| 4 | Rent Sysvar | ❌ | ❌ |

---

## Instruction Data Layout

```
u8     discriminator (39)
Pubkey tokenMint
```

---

## TypeScript Builder

```ts
function createInitializeUserAccountInstruction(
  programId: PublicKey,
  user: PublicKey,
  userAccountPda: PublicKey,
  tokenMint: PublicKey
): TransactionInstruction {
  const data = Buffer.concat([
    Buffer.from([39]), // INITIALIZE_USER_ACCOUNT_TAG
    tokenMint.toBuffer(),
  ]);

  return new TransactionInstruction({
    keys: [
      { pubkey: user, isSigner: true, isWritable: true },
      { pubkey: userAccountPda, isSigner: false, isWritable: true },
      { pubkey: tokenMint, isSigner: false, isWritable: false },
      { pubkey: SystemProgram.programId, isSigner: false, isWritable: false },
      { pubkey: SYSVAR_RENT_PUBKEY, isSigner: false, isWritable: false },
    ],
    programId,
    data,
  });
}
```

---

# Example Flow (From Trading Test)

```ts
const [userAccountPda] = findUserAccountAddress(
  PROGRAM_ID,
  trader.publicKey,
  mintKeypair.publicKey
);

const initUserAccountIx = createInitializeUserAccountInstruction(
  PROGRAM_ID,
  trader.publicKey,
  userAccountPda,
  mintKeypair.publicKey
);

const tx = new Transaction().add(initUserAccountIx);

await sendAndConfirmTransaction(
  connection,
  tx,
  [trader],
  { commitment: 'confirmed' }
);
```

---

# What Happens On-Chain

After execution:

- PDA account is created
- Owner = program
- `is_initialized = true`
- Balances = 0
- `active_orders = 0`
- `active_limit_orders = 0`
- `orders = []`
- `limit_orders = []`
- `token_accounts = []`

This account becomes the central state container for:

- Deposits
- Margin tracking
- Funding payments
- Order tracking
- Position management

---

# Important Notes

- A user must have one `UserAccount` per settlement token mint.
- Re-initializing an existing PDA will fail.
- Deposit must occur **after** initialization.
- All future trading instructions require this PDA.
