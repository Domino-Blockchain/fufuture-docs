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

## 1️⃣ Pool PDA

```
["pool", token_mint, underlying_name]
```

```ts
function findPoolAddress(
  programId: PublicKey,
  tokenMint: PublicKey,
  underlyingName: string
): [PublicKey, number] {
  return PublicKey.findProgramAddressSync(
    [
      Buffer.from('pool'),
      tokenMint.toBuffer(),
      Buffer.from(underlyingName, 'utf-8')
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
| 1 | Pool PDA | ❌ | ✅ |
| 2 | Makers PDA | ❌ | ✅ |
| 3 | LiquidityMarkets PDA | ❌ | ✅ |
| 4 | MatchIds PDA | ❌ | ✅ |
| 5 | ShareHolders PDA | ❌ | ✅ |
| 6 | UserDeals PDA | ❌ | ✅ |
| 7 | Positions PDA | ❌ | ✅ |
| 8 | Keepers PDA | ❌ | ✅ |
| 9 | Token Mint | ❌ | ❌ |
| 10 | Risk Fund | ❌ | ❌ |
| 11 | Program ID | ❌ | ❌ |
| 12 | Program ID | ❌ | ❌ |
| 13 | Program ID | ❌ | ❌ |
| 14 | System Program | ❌ | ❌ |
| 15 | Rent Sysvar | ❌ | ❌ |

⚠️ **Account ordering must match exactly.**

---

# Instruction Data Layout

```
u8      discriminator (19)
u32     underlying_name length
bytes   underlying_name
Pubkey  risk_fund
u8      token_decimals
```

---

# TypeScript Instruction Builder

```ts
function createInitializePublicPoolInstruction(
  programId: PublicKey,
  authority: PublicKey,
  poolPda: PublicKey,
  poolPdas: PoolPDAs,
  tokenMint: PublicKey,
  riskFund: PublicKey,
  underlyingName: string,
  tokenDecimals: number
): TransactionInstruction {

  const nameBytes = Buffer.from(underlyingName, 'utf-8');
  const nameLengthBuffer = Buffer.alloc(4);
  nameLengthBuffer.writeUInt32LE(nameBytes.length, 0);

  const data = Buffer.concat([
    Buffer.from([19]),
    nameLengthBuffer,
    nameBytes,
    riskFund.toBuffer(),
    Buffer.from([tokenDecimals]),
  ]);

  return new TransactionInstruction({
    keys: [
      { pubkey: authority, isSigner: true, isWritable: true },
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
