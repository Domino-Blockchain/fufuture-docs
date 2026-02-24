---
id: withdraw
title: Withdraw (Trader Collateral)
sidebar_position: 10
---

# Withdraw

This instruction allows a trader to withdraw collateral from their `UserAccount` back to their wallet token account.

**Instruction:** `Withdraw`  
**Discriminator:** `1`

---

# Overview

The `Withdraw` instruction:

- Debits trader collateral inside `UserAccount`
- Transfers tokens from pool vault → trader token account
- Validates margin safety (no under-collateralization)
- Updates protocol accounting

Collateral is stored as `u128` in **E18 precision format**.

If the trader has open positions, the withdraw logic must validate margin health across the specified trading pairs.

---

# Account Order (Strict)

| Index | Account | Signer | Writable |
|-------|----------|----------|-----------|
| 0 | Trader (user) | ✅ | ❌ |
| 1 | UserAccount PDA | ❌ | ✅ |
| 2 | Pool Token Account (vault source) | ❌ | ✅ |
| 3 | Trader Token Account (destination) | ❌ | ✅ |
| 4 | Token Program | ❌ | ❌ |
| 5 | Program Authority PDA (`["authority"]`) | ❌ | ❌ |
| 6 | Token Mint | ❌ | ❌ |
| 7 | ConfigCore PDA | ❌ | ❌ |

⚠️ Account ordering must exactly match the on-chain processor.

---

# Instruction Data Layout

```

u8          discriminator (1)
u128        amount (E18 precision)
Vec<String> symbols

```

### Borsh Encoding of `Vec<String>`

```

u32     vector length
repeat:
u32   string length
[u8]  UTF-8 bytes

````

- `amount` is serialized little-endian (16 bytes)
- `symbols` is required for margin validation when open positions exist
- Pass an empty vector if no open positions

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
````

---

# TypeScript Instruction Builder

```ts
const WITHDRAW_TAG = 1;

function createWithdrawInstruction(
  programId: PublicKey,
  user: PublicKey,
  userAccountPda: PublicKey,
  poolTokenAccount: PublicKey,
  userTokenAccount: PublicKey,
  programAuthority: PublicKey,
  tokenMint: PublicKey,
  configPda: PublicKey,
  amount: string,
  symbols: string[]
): TransactionInstruction {

  const vecLenBuf = Buffer.alloc(4);
  vecLenBuf.writeUInt32LE(symbols.length, 0);

  const symbolBufs = symbols.flatMap(s => {
    const strBytes = Buffer.from(s, 'utf-8');
    const lenBuf = Buffer.alloc(4);
    lenBuf.writeUInt32LE(strBytes.length, 0);
    return [lenBuf, strBytes];
  });

  const data = Buffer.concat([
    Buffer.from([WITHDRAW_TAG]),
    writeU128LE(amount),
    vecLenBuf,
    ...symbolBufs,
  ]);

  return new TransactionInstruction({
    keys: [
      { pubkey: user,             isSigner: true,  isWritable: false },
      { pubkey: userAccountPda,   isSigner: false, isWritable: true  },
      { pubkey: poolTokenAccount, isSigner: false, isWritable: true  },
      { pubkey: userTokenAccount, isSigner: false, isWritable: true  },
      { pubkey: TOKEN_PROGRAM_ID, isSigner: false, isWritable: false },
      { pubkey: programAuthority, isSigner: false, isWritable: false },
      { pubkey: tokenMint,        isSigner: false, isWritable: false },
      { pubkey: configPda,        isSigner: false, isWritable: false },
    ],
    programId,
    data,
  });
}
```

---

# Execution Flow

1. Trader signs withdraw request
2. Program validates:

   * UserAccount is initialized
   * Withdraw amount ≤ available collateral
   * Margin remains healthy after withdrawal
3. Collateral is decremented inside `UserAccount`
4. SPL transfer:

   * From pool vault (owned by program authority)
   * To trader token account

---

# Example Usage

```ts
const WITHDRAW_AMOUNT = "500000000000000000"; // 0.5 tokens in E18

const withdrawSymbols: string[] = []; // include pairs if positions exist

const withdrawIx = createWithdrawInstruction(
  PROGRAM_ID,
  trader.publicKey,
  userAccountPda,
  poolTokenAccount,
  traderTokenAccount,
  programAuthority,
  mintKeypair.publicKey,
  configCorePDA,
  WITHDRAW_AMOUNT,
  withdrawSymbols
);

const withdrawTx = new Transaction().add(withdrawIx);

const withdrawSig = await sendAndConfirmTransaction(
  connection,
  withdrawTx,
  [trader],
  { commitment: 'confirmed' }
);

console.log("Withdraw successful:", withdrawSig);
```

---

# On-Chain Effects

After execution:

* `UserAccount.collateral` decreases (u128)
* Pool vault balance decreases
* Trader token balance increases
* Margin requirements remain satisfied

If withdrawal would violate maintenance margin, the transaction reverts.

---

# Precision Model

* SPL Token → 9 decimals (example)
* Internal protocol math → E18 precision
* Always convert correctly before sending withdraw amount

Example:

```
0.5 token → "500000000000000000"
```

Incorrect E18 formatting will break margin accounting.

Withdraw is permitted only if resulting collateral maintains required leverage and margin thresholds.

