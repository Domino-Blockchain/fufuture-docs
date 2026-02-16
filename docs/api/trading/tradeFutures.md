# TradeFutures

This instruction opens a futures position (e.g. LONG or SHORT) and creates:

- A new `Deal` PDA
- A `Position` PDA (if not already existing)
- Updates orderbook state
- Updates pool + user state

---

## Instruction Discriminator

```ts
const TRADE_FUTURES_TAG = 7;
````

---

## Helper: writeU128LE

```ts
function writeU128LE(value: string): Buffer {
  const bn = BigInt(value);
  const buf = Buffer.alloc(16);
  let remaining = bn;

  for (let i = 0; i < 16; i++) {
    buf[i] = Number(remaining & BigInt(0xff));
    remaining >>= BigInt(8);
  }

  return buf;
}
```

---

## TradeFutures Instruction Builder

```ts
function createTradeFuturesInstruction(
  programId: PublicKey,
  keeper: PublicKey,
  taker: PublicKey,
  userAccountPda: PublicKey,
  assetPda: PublicKey,
  poolPda: PublicKey,
  poolPdas: PoolPDAs,
  configPda: PublicKey,
  orderbookState: PublicKey,
  currentDealId: number,
  pair: string,
  amount: string,
  price: string,
  orderType: number,
  direction: number,
  goodTill: string,
  rewardGas: string
): TransactionInstruction {

  const pairBytes = Buffer.from(pair, 'utf-8');
  const pairLengthBuffer = Buffer.alloc(4);
  pairLengthBuffer.writeUInt32LE(pairBytes.length, 0);

  const goodTillBuf = Buffer.alloc(8);
  goodTillBuf.writeBigInt64LE(BigInt(goodTill), 0);

  const rewardGasBuf = Buffer.alloc(8);
  rewardGasBuf.writeBigUInt64LE(BigInt(rewardGas), 0);

  const data = Buffer.concat([
    Buffer.from([TRADE_FUTURES_TAG]),
    pairLengthBuffer,
    pairBytes,
    writeU128LE(amount),
    writeU128LE(price),
    Buffer.from([orderType]),
    Buffer.from([direction]),
    goodTillBuf,
    taker.toBuffer(),
    rewardGasBuf,
  ]);

  const [dealPda] = findDealAddress(programId, currentDealId);
  const [positionPda] = findPositionAddress(programId, taker, pair, direction);

  return new TransactionInstruction({
    keys: [
      { pubkey: keeper, isSigner: true, isWritable: true },               // 0
      { pubkey: assetPda, isSigner: false, isWritable: false },           // 1
      { pubkey: configPda, isSigner: false, isWritable: true },           // 2
      { pubkey: taker, isSigner: false, isWritable: false },              // 3
      { pubkey: userAccountPda, isSigner: false, isWritable: true },      // 4
      { pubkey: poolPda, isSigner: false, isWritable: true },             // 5
      { pubkey: poolPdas.makers, isSigner: false, isWritable: true },     // 6
      { pubkey: poolPdas.liquidityMarkets, isSigner: false, isWritable: true }, // 7
      { pubkey: poolPdas.positions, isSigner: false, isWritable: true },  // 8
      { pubkey: poolPdas.userDeals, isSigner: false, isWritable: true },  // 9
      { pubkey: orderbookState, isSigner: false, isWritable: true },      // 10
      { pubkey: dealPda, isSigner: false, isWritable: true },             // 11
      { pubkey: positionPda, isSigner: false, isWritable: true },         // 12
      { pubkey: SystemProgram.programId, isSigner: false, isWritable: false }, // 13
      { pubkey: SYSVAR_RENT_PUBKEY, isSigner: false, isWritable: false }, // 14
      { pubkey: SYSVAR_CLOCK_PUBKEY, isSigner: false, isWritable: false }, // 15
      { pubkey: poolPdas.matchIds, isSigner: false, isWritable: true },   // 16
      { pubkey: poolPdas.keepers, isSigner: false, isWritable: false },   // 17
    ],
    programId,
    data,
  });
}
```

---

## Required Account Order (Critical)

The account order must match the Rust `open_order` / `trade_futures` processor exactly:

| Index | Account               |
| ----- | --------------------- |
| 0     | keeper (signer)       |
| 1     | asset PDA             |
| 2     | config PDA            |
| 3     | taker                 |
| 4     | user account PDA      |
| 5     | pool PDA              |
| 6     | makers PDA            |
| 7     | liquidity markets PDA |
| 8     | positions PDA         |
| 9     | user deals PDA        |
| 10    | orderbook state       |
| 11    | deal PDA              |
| 12    | position PDA          |
| 13    | system program        |
| 14    | rent sysvar           |
| 15    | clock sysvar          |
| 16    | match ids PDA         |
| 17    | keepers PDA           |

If the order differs, deserialization will fail or produce invalid bool / UnsupportedAsset errors.

---

## Data Layout (Binary Encoding)    

The instruction data is encoded as:

```
u8      instruction_tag (7)
u32     pair_length
bytes   pair_string
u128    amount
u128    price
u8      order_type
u8      direction
i64     good_till
Pubkey  taker
u64     reward_gas
```

All numeric values must be little-endian.

---

