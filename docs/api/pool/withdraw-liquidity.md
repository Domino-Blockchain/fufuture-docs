---
id: withdraw-public-liquidity
title: Withdraw Public Liquidity
sidebar_position: 9
---

# Withdraw Public Liquidity

This instruction allows a liquidity provider (LP) to withdraw previously deposited liquidity from a public pool.

**Instruction:** `WithdrawPublicLiquidity`  
**Discriminator:** `21`

---

# Overview

Withdraws liquidity from an initialized public pool.

- Burns LP share accounting
- Transfers underlying tokens back to provider
- Updates pool exposure state
- Updates shareholder state
- Adjusts makers & position accounting

This is the reverse operation of `AddPublicLiquidity`.

---

# Account Order (Strict)

| Index | Account | Signer | Writable |
|-------|----------|----------|-----------|
| 0 | Liquidity Provider | ✅ | ✅ |
| 1 | Pool PDA | ❌ | ✅ |
| 2 | Makers PDA | ❌ | ✅ |
| 3 | ShareHolders PDA | ❌ | ✅ |
| 4 | Provider Token Account (destination) | ❌ | ✅ |
| 5 | Pool Token Account (source) | ❌ | ✅ |
| 6 | Positions PDA | ❌ | ✅ |
| 7 | ConfigCore PDA | ❌ | ❌ |
| 8 | Token Program | ❌ | ❌ |
| 9 | System Program | ❌ | ❌ |

⚠️ Account ordering must match the on-chain program exactly.

---

# Instruction Data Layout

```
u8   discriminator (21)
u64  lp_amount
```

Where:

- `lp_amount` = amount of LP shares to burn (token decimals)

---

# TypeScript Instruction Builder

```ts
const WITHDRAW_PUBLIC_LIQUIDITY_TAG = 21;

function createWithdrawPublicLiquidityInstruction(
  programId: PublicKey,
  provider: PublicKey,
  poolPda: PublicKey,
  makersPda: PublicKey,
  shareHoldersPda: PublicKey,
  providerTokenAccount: PublicKey,
  poolTokenAccount: PublicKey,
  positionsPda: PublicKey,
  configCorePda: PublicKey,
  amount: number
): TransactionInstruction {

  const amountBuffer = Buffer.alloc(8);
  amountBuffer.writeBigUInt64LE(BigInt(amount), 0);

  const data = Buffer.concat([
    Buffer.from([WITHDRAW_PUBLIC_LIQUIDITY_TAG]),
    amountBuffer,
  ]);

  return new TransactionInstruction({
    keys: [
      { pubkey: provider, isSigner: true, isWritable: true },
      { pubkey: poolPda, isSigner: false, isWritable: true },
      { pubkey: makersPda, isSigner: false, isWritable: true },
      { pubkey: shareHoldersPda, isSigner: false, isWritable: true },
      { pubkey: providerTokenAccount, isSigner: false, isWritable: true },
      { pubkey: poolTokenAccount, isSigner: false, isWritable: true },
      { pubkey: positionsPda, isSigner: false, isWritable: true },
      { pubkey: configCorePda, isSigner: false, isWritable: false },
      { pubkey: TOKEN_PROGRAM_ID, isSigner: false, isWritable: false },
      { pubkey: SystemProgram.programId, isSigner: false, isWritable: false },
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
4. Derive positions PDA
5. Provide provider token account (destination)
6. Provide pool vault token account (source)
7. Build withdraw instruction
8. Send transaction

---

# Example Usage

```ts
const withdrawLiquidityIx = createWithdrawPublicLiquidityInstruction(
  PROGRAM_ID,
  liquidityProvider.publicKey,
  poolPda,
  makersPda,
  shareHoldersPda,
  providerTokenAccount,
  poolTokenAccount,
  positionsPda,
  configCorePda,
  WITHDRAW_AMOUNT
);

const withdrawTx = new Transaction().add(withdrawLiquidityIx);

const withdrawSig = await sendAndConfirmTransaction(
  connection,
  withdrawTx,
  [liquidityProvider],
  { commitment: 'confirmed' }
);
```

---

# On-Chain Effects

After successful execution:

- LP share balance reduced
- Provider receives underlying tokens
- Pool vault balance decreases
- ShareHolders accounting updated
- Maker exposure recalculated
- Positions state adjusted if necessary

---


