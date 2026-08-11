---
id: deposit-collateral
title: Deposit (Trader Collateral)
sidebar_position: 10
---

# Deposit (Trader Collateral)

This instruction allows a trader to deposit collateral into their `UserAccount` for trading futures.

**Instruction:** `Deposit`  
**Discriminator:** `0`

---

# Overview

The `Deposit` instruction:

- Transfers tokens from trader → pool vault
- Credits trader collateral inside `UserAccount`
- Updates protocol accounting
- Enables futures trading

Collateral is written as `u128` in **E18 precision format**.

---

# Account Order (Strict)

| Index | Account | Signer | Writable |
|-------|----------|----------|-----------|
| 0 | Trader (user) | ✅ | ✅ |
| 1 | UserAccount PDA | ❌ | ✅ |
| 2 | Trader Token Account (source) | ❌ | ✅ |
| 3 | Pool Token Account (vault) | ❌ | ✅ |
| 4 | Token Program | ❌ | ❌ |
| 5 | ConfigCore PDA | ❌ | ✅ |
| 6 | Token Mint | ❌ | ❌ |

⚠️ Account ordering must exactly match the on-chain processor.

---

## Security Constraints

- Account 4 must be the real SPL Token Program.
- Account 6 must be a real SPL mint.
- Account 1 must be the canonical program-owned UserAccount PDA for
  `(trader, token mint)`, and its stored user must equal the signer.
- Account 5 must be the canonical program-owned ConfigCore PDA.
- Account 3 must be an SPL token account for account 6 whose token authority
  is the program `["authority"]` PDA.

Caller-owned or otherwise stand-in vaults are rejected before the transfer and
cannot be used to create protocol collateral from a self-transfer.

# Instruction Data Layout

```
u8     discriminator (0)
u128   amount (E18 precision)
```

- `amount` is serialized little-endian (16 bytes)
- Must match Rust `u128` layout exactly

---

# U128 Encoding Helper

```ts
function writeU128LE(value: string): Buffer {
  const bn = BigInt(value);
  const buf = Buffer.alloc(16);
  let remaining = bn;

  for (let i = 0; i < 16; i++) {
    buf[i] = Number(remaining & BigInt(0xff));
    remaining = remaining >> BigInt(8);
  }

  return buf;
}
```

---

# TypeScript Instruction Builder

```ts
const DEPOSIT_TAG = 0;

function createDepositInstruction(
  programId: PublicKey,
  user: PublicKey,
  userAccountPda: PublicKey,
  userTokenAccount: PublicKey,
  poolTokenAccount: PublicKey,
  configPda: PublicKey,
  tokenMint: PublicKey,
  amount: string
): TransactionInstruction {

  const data = Buffer.concat([
    Buffer.from([DEPOSIT_TAG]),
    writeU128LE(amount),
  ]);

  return new TransactionInstruction({
    keys: [
      { pubkey: user, isSigner: true, isWritable: true },
      { pubkey: userAccountPda, isSigner: false, isWritable: true },
      { pubkey: userTokenAccount, isSigner: false, isWritable: true },
      { pubkey: poolTokenAccount, isSigner: false, isWritable: true },
      { pubkey: TOKEN_PROGRAM_ID, isSigner: false, isWritable: false },
      { pubkey: configPda, isSigner: false, isWritable: true },
      { pubkey: tokenMint, isSigner: false, isWritable: false },
    ],
    programId,
    data,
  });
}
```

---

# Execution Flow

1. Trader initializes `UserAccount` (if not already)
2. Trader approves token transfer
3. Deposit instruction:
   - Moves tokens → pool vault
   - Increments user collateral balance
   - Updates config accounting

---

# Example Usage

```ts
const depositIx = createDepositInstruction(
  PROGRAM_ID,
  trader.publicKey,
  userAccountPda,
  traderTokenAccount,
  poolTokenAccount,
  configCorePDA,
  mintKeypair.publicKey,
  "1000000000000000000" // 1 token in E18
);

const depositTx = new Transaction().add(depositIx);

const depositSig = await sendAndConfirmTransaction(
  connection,
  depositTx,
  [trader],
  { commitment: 'confirmed' }
);

console.log("Deposit successful:", depositSig);
```

---

# On-Chain Effects

After execution:

- Trader token balance decreases
- Pool vault balance increases
- `UserAccount.collateral` increases (u128)
- Trader can now open leveraged positions

---

# Precision Model

- SPL Token → 9 decimals (example)
- Internal protocol math → E18 precision
- Always convert correctly before sending deposit amount

Example:

```
1 token → "1000000000000000000"
```

Mismatch between SPL decimals and E18 math will cause incorrect margin accounting.

Deposit is mandatory before any futures position can be opened.
