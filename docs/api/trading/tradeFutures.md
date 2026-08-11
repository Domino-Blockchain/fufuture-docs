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
    currentOrderCount: bigint,
    pair: string,
    amount: string,
    price: string,
    orderType: number,
    direction: number,
    goodTill: string,
    rewardGas: string,
    existingDealIds: bigint[] = []
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

    const [dealPda] = findDealAddress(programId, currentOrderCount);
    const [positionPda] = findPositionAddress(programId, taker, pair, direction);

    console.log(`\nTradeFutures PDA Debug:`);
    console.log(`  Deal PDA (order_count=${currentOrderCount}): ${dealPda.toBase58()}`);
    console.log(`  Position PDA: ${positionPda.toBase58()}`);
    if (existingDealIds.length > 0) {
        console.log(`  Appending ${existingDealIds.length} existing deal PDAs: [${existingDealIds.join(', ')}]`);
    }

    return new TransactionInstruction({
        keys: [
            {pubkey: keeper, isSigner: true, isWritable: true},              // 0
            {pubkey: assetPda, isSigner: false, isWritable: false},          // 1
            {pubkey: configPda, isSigner: false, isWritable: true},          // 2
            {pubkey: taker, isSigner: false, isWritable: false},             // 3
            {pubkey: userAccountPda, isSigner: false, isWritable: true},     // 4
            {pubkey: poolPda, isSigner: false, isWritable: true},            // 5
            {pubkey: poolPdas.makers, isSigner: false, isWritable: true},    // 6
            {pubkey: poolPdas.liquidityMarkets, isSigner: false, isWritable: true}, // 7
            {pubkey: poolPdas.positions, isSigner: false, isWritable: true}, // 8
            {pubkey: poolPdas.userDeals, isSigner: false, isWritable: true}, // 9
            {pubkey: orderbookState, isSigner: false, isWritable: true},     // 10
            {pubkey: dealPda, isSigner: false, isWritable: true},            // 11
            {pubkey: positionPda, isSigner: false, isWritable: true},        // 12
            {pubkey: SystemProgram.programId, isSigner: false, isWritable: false}, // 13
            {pubkey: SYSVAR_RENT_PUBKEY, isSigner: false, isWritable: false},      // 14
            {pubkey: SYSVAR_CLOCK_PUBKEY, isSigner: false, isWritable: false},     // 15
            {pubkey: poolPdas.matchIds, isSigner: false, isWritable: true},  // 16
            {pubkey: poolPdas.keepers, isSigner: false, isWritable: false},  // 17
            // Existing deal PDAs for sort_and_insert_deal (accounts[18+])
            ...existingDealIds.map(id => ({
                pubkey: findDealAddress(programId, id)[0],
                isSigner: false,
                isWritable: false,
            })),
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
| 18+   | existing deal PDAs (RO) |


If the order differs, deserialization will fail or produce invalid bool / UnsupportedAsset errors.

---

## Token Routing Security Contract

The Batch 1 fund-loss hardening does not change instruction data, account count,
or account order. Any open-order path that reaches pool matching must now satisfy
all of the following bindings before protocol state is mutated:

- the token program is the canonical SPL Token Program;
- the token mint is a valid SPL Mint and equals the pool's `token_address`;
- the pool vault is an SPL Token Account for that mint whose token authority is
  the program `['authority']` PDA;
- the risk-fund account is the exact SPL Token Account stored in
  `pool.risk_fund_addr`, has the same mint and program authority, and is distinct
  from the pool vault;
- when the fee-vault PDA is supplied, it is also an SPL Token Account for the
  pool mint under the program authority.

The same pool-vault, risk-fund, mint, program-authority, and token-program
contract applies to `ClosePosition` / trigger-close settlement. Clients must not
derive `ATA(pool.risk_fund_addr, mint)`: `risk_fund_addr` already is the exact
dedicated token-account public key.

---

## Helper

When opening a new order on an existing position:

1. Fetch the Position PDA.

2. Parse its deals: Vec field.

3. Pass those deal IDs to createTradeFuturesInstruction.

Example helper:

```ts
async function getPositionDealIds(
    connection: Connection,
    programId: PublicKey,
    taker: PublicKey,
    pair: string,
    direction: number
): Promise<bigint[]> {
    const [positionPda] = findPositionAddress(programId, taker, pair, direction);
    const accountInfo = await connection.getAccountInfo(positionPda);
    if (!accountInfo) return []; // No position yet

    try {
        // Position Borsh layout:
        // is_initialized: 1
        // user: 32
        // pair: 4 + N (string)
        // direction: 1
        // value: 16
        // size: 16
        // margin: 16
        // freeze: 16
        // deals: 4 (vec length) + 8 * count (u64 each)
        // bump: 1
        const data = accountInfo.data;
        let offset = 0;
        
        offset += 1; // is_initialized
        offset += 32; // user pubkey
        
        // Read pair string (4-byte length prefix)
        const pairLen = data.readUInt32LE(offset);
        offset += 4 + pairLen;
        
        offset += 1;  // direction
        offset += 16; // value
        offset += 16; // size
        offset += 16; // margin
        offset += 16; // freeze
        
        // Read deals vec
        const dealsCount = data.readUInt32LE(offset);
        offset += 4;
        
        const dealIds: bigint[] = [];
        for (let i = 0; i < dealsCount; i++) {
            const dealId = data.readBigUInt64LE(offset);
            offset += 8;
            dealIds.push(dealId);
        }
        
        console.log(`  Existing position deals: [${dealIds.join(', ')}]`);
        return dealIds;
    } catch (e) {
        console.warn('Failed to parse position deal IDs:', e);
        return [];
    }
}
```

This reads the Vec from the Position account and returns deal IDs that must be appended as remaining accounts.

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

