---
id: initialize-public-pool
title: Initialize Public Pool
sidebar_position: 7
---

# Initialize Public Pool

This instruction creates a **unified public liquidity pool** along with all required auxiliary state accounts.

It must be executed before:

- Adding liquidity
- Matching orders against the pool
- Executing liquidations involving the pool
- Recording maker deals

---

# Instruction Overview

**Discriminator:** `19`  
**Instruction:** `InitializePublicPool`

---

# PDA Structure

The pool system consists of **1 main PDA + 7 auxiliary PDAs**.

---

## 0️⃣ PoolList PDA

Stores all initialized settlement-token pools.

```
["pool_list"]
```

```ts
function findPoolListAddress(
  programId: PublicKey
): [PublicKey, number] {
  return PublicKey.findProgramAddressSync(
    [Buffer.from('pool_list')],
    programId
  );
}
```

---


## 1️⃣ Pool PDA

```
["pool", token_mint]
```

```ts
function findPoolAddress(
  programId: PublicKey,
  tokenMint: PublicKey,
): [PublicKey, number] {
  return PublicKey.findProgramAddressSync(
    [
      Buffer.from('pool'),
      tokenMint.toBuffer()
    ],
    programId
  );
}
```

---

## 2️⃣ Auxiliary PDAs

Each auxiliary account is derived from the pool PDA.

---

### Makers

```
["makers", pool_pda]
```

---

### Liquidity Markets

```
["liquidity_markets", pool_pda]
```

---

### Match IDs

```
["match_ids", pool_pda]
```

---

### User Deals

```
["user_deals", pool_pda]
```

---

### Positions

```
["position", pool_pda]
```

---

### Share Holders

```
["share_holders", pool_pda]
```

---

### Keepers

```
["keepers", pool_pda]
```

---

# Required Accounts (Strict Order)

| Index | Account | Signer | Writable |
|--------|----------|----------|-----------|
| 0 | Authority (payer) | ✅ | ✅ |
| 1 | PoolList PDA | ❌ | ✅ |
| 2 | Pool PDA | ❌ | ✅ |
| 3 | Makers PDA | ❌ | ✅ |
| 4 | LiquidityMarkets PDA | ❌ | ✅ |
| 5 | MatchIds PDA | ❌ | ✅ |
| 6 | ShareHolders PDA | ❌ | ✅ |
| 7 | UserDeals PDA | ❌ | ✅ |
| 8 | Positions PDA | ❌ | ✅ |
| 9 | Keepers PDA | ❌ | ✅ |
| 10 | Token Mint | ❌ | ❌ |
| 11 | Risk Fund | ❌ | ❌ |
| 12 | Program ID | ❌ | ❌ |
| 13 | Program ID | ❌ | ❌ |
| 14 | Program ID | ❌ | ❌ |
| 15 | System Program | ❌ | ❌ |
| 16 | Rent Sysvar | ❌ | ❌ |

⚠️ **Account ordering must match exactly.**

---

## Risk Fund Account Contract

`risk_fund` is the public key of the **exact SPL Token Account stored in the
pool state**. It is not a wallet address and clients must not derive
`ATA(risk_fund, token_mint)`.

Before initializing a pool, create a dedicated token account whose:

- outer account owner is the SPL Token Program;
- mint is the pool settlement mint;
- token authority is the program `["authority"]` PDA;
- address is different from the pool vault.

Pass that token account public key at account index 11 and in the instruction
data. Open/close and fee-routing instructions must later pass the same exact
account.

# Instruction Data Layout

```
u8      discriminator (19)
Pubkey  risk_fund
u8      token_decimals
```

---

# TypeScript Instruction Builder

```ts
function createInitializePublicPoolInstruction(
  programId: PublicKey,
  authority: PublicKey,
  poolListPda: PublicKey,
  poolPda: PublicKey,
  poolPdas: PoolPDAs,
  tokenMint: PublicKey,
  riskFund: PublicKey,
  tokenDecimals: number
): TransactionInstruction {

  const data = Buffer.concat([
    Buffer.from([19]),
    riskFund.toBuffer(),
    Buffer.from([tokenDecimals]),
  ]);

  return new TransactionInstruction({
    keys: [
      { pubkey: authority, isSigner: true, isWritable: true },
      { pubkey: poolListPda, isSigner: false, isWritable: true },
      { pubkey: poolPda, isSigner: false, isWritable: true },
      { pubkey: poolPdas.makers, isSigner: false, isWritable: true },
      { pubkey: poolPdas.liquidityMarkets, isSigner: false, isWritable: true },
      { pubkey: poolPdas.matchIds, isSigner: false, isWritable: true },
      { pubkey: poolPdas.shareHolders, isSigner: false, isWritable: true },
      { pubkey: poolPdas.userDeals, isSigner: false, isWritable: true },
      { pubkey: poolPdas.positions, isSigner: false, isWritable: true },
      { pubkey: poolPdas.keepers, isSigner: false, isWritable: true },
      { pubkey: tokenMint, isSigner: false, isWritable: false },
      { pubkey: riskFund, isSigner: false, isWritable: false },
      { pubkey: programId, isSigner: false, isWritable: false },
      { pubkey: programId, isSigner: false, isWritable: false },
      { pubkey: programId, isSigner: false, isWritable: false },
      { pubkey: SystemProgram.programId, isSigner: false, isWritable: false },
      { pubkey: SYSVAR_RENT_PUBKEY, isSigner: false, isWritable: false },
    ],
    programId,
    data,
  });
}
```

---

# Execution Flow

1. Derive pool PDA
2. Derive 7 auxiliary PDAs
3. Check if pool already exists
4. Build instruction
5. Send transaction
6. Verify all 8 accounts were created

---

# What Gets Created

After successful execution:

- Pool account
- Makers account
- LiquidityMarkets account
- MatchIds account
- UserDeals account
- Positions account
- ShareHolders account
- Keepers account

All accounts are program-owned and rent-exempt.

---

# Result

A fully initialized public liquidity pool ready for:

- Liquidity provisioning
- Order matching
- Position management
- Liquidation handling
