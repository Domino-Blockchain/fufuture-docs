---
id: add-public-liquidity
title: Add Public Liquidity
sidebar_position: 8
---

# Add Public Liquidity

This instruction allows a liquidity provider (LP) to deposit tokens into an initialized public pool in exchange for pool share accounting.

**Instruction:** `AddPublicLiquidity`  
**Discriminator:** `20`

---

# Overview

Adds liquidity to an existing public pool.

- Transfers tokens from the provider to the pool vault
- Updates pool accounting
- Updates shareholder state
- Updates maker exposure tracking

---

# PDA Requirements

The pool must already be initialized.

You will need:

- Pool PDA
- Makers PDA
- ShareHolders PDA
- Positions PDA
- Config PDA

---

# Account Order (Strict)

| Index | Account | Signer | Writable |
|-------|----------|----------|-----------|
| 0 | Liquidity Provider | ✅ | ❌ |
| 1 | Pool PDA | ❌ | ✅ |
| 2 | Makers PDA | ❌ | ✅ |
| 3 | ShareHolders PDA | ❌ | ✅ |
| 4 | Provider Token Account (source) | ❌ | ✅ |
| 5 | Pool Token Account (destination) | ❌ | ✅ |
| 6 | Positions PDA | ❌ | ✅ |
| 7 | Config PDA | ❌ | ❌ |
| 8 | Token Program | ❌ | ❌ |

⚠️ Account ordering must match exactly.

---

## Security Constraints

- The final account must be the real SPL Token Program.
- The pool and Makers accounts must be the canonical program-owned accounts.
- The pool vault must be owned by the SPL Token Program, use the pool's
  settlement mint, and have the program `["authority"]` PDA as token authority.

A caller-supplied program that reports a successful CPI without moving tokens
is rejected before LP shares are credited.

# Instruction Data Layout

```
u8   discriminator (20)
u64  amount (token decimals)
```

---

# TypeScript Instruction Builder

```ts
const ADD_PUBLIC_LIQUIDITY_TAG = 20;

function createAddPublicLiquidityInstruction(
  programId: PublicKey,
  provider: PublicKey,
  poolPda: PublicKey,
  makersPda: PublicKey,
  shareHoldersPda: PublicKey,
  providerTokenAccount: PublicKey,
  poolTokenAccount: PublicKey,
  positionsPda: PublicKey,
  configPda: PublicKey,
  amount: bigint
): TransactionInstruction {

  const amountBuffer = Buffer.alloc(8);
  amountBuffer.writeBigUInt64LE(amount, 0);

  const data = Buffer.concat([
    Buffer.from([ADD_PUBLIC_LIQUIDITY_TAG]),
    amountBuffer,
  ]);

  return new TransactionInstruction({
    keys: [
      { pubkey: provider, isSigner: true, isWritable: false },
      { pubkey: poolPda, isSigner: false, isWritable: true },
      { pubkey: makersPda, isSigner: false, isWritable: true },
      { pubkey: shareHoldersPda, isSigner: false, isWritable: true },
      { pubkey: providerTokenAccount, isSigner: false, isWritable: true },
      { pubkey: poolTokenAccount, isSigner: false, isWritable: true },
      { pubkey: positionsPda, isSigner: false, isWritable: true },
      { pubkey: configPda, isSigner: false, isWritable: false },
      { pubkey: TOKEN_PROGRAM_ID, isSigner: false, isWritable: false },
    ],
    programId,
    data,
  });
}
```

---

# Execution Flow

1. Derive pool PDA
2. Derive makers PDA
3. Derive share holders PDA
4. Fetch or derive positions PDA
5. Provide provider token account
6. Provide pool vault token account
7. Build instruction
8. Send transaction

---

# On-Chain Effects

After successful execution:

- Provider tokens are transferred into pool vault
- Pool liquidity increases
- ShareHolders account updated
- Makers exposure updated
- Positions accounting adjusted

---

# Example Usage

```ts
const instruction = createAddPublicLiquidityInstruction(
  PROGRAM_ID,
  authority.publicKey,
  poolPda,
  makersPda,
  shareHoldersPda,
  providerTokenAccount,
  poolVaultAccount,
  positionsPda,
  configPda,
  1_000_000_000n // Example: 1 token with 9 decimals
);

const tx = new Transaction().add(instruction);

await sendAndConfirmTransaction(
  connection,
  tx,
  [authority],
  { commitment: 'confirmed' }
);
```

---

# Result

Liquidity provider is now part of the pool and entitled to:

- Trading fee share
- Funding payments
- PnL exposure
- Liquidation proceeds (if applicable)
