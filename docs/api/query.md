---
id: queries
title: Query
sidebar_position: 5
---

# Query API Reference

All queries are **read-only RPC calls** using:

```
connection.getAccountInfo(PDA)
```

No instruction execution is required.

All accounts are Borsh-encoded and must be decoded using the exact schemas defined earlier.

---

# Configuration Queries

These queries read protocol-level configuration from the 4 Config PDAs.

---

## ConfigCore

### PDA

```
["config"]
```

### Purpose

Global protocol configuration:

- Owner
- Authority
- Fee parameters
- Leverage limits
- Margin settings
- Order counters
- System state (paused flag)

### Query Example

```ts
const [configCorePDA] = findConfigCoreAddress(PROGRAM_ID);
const accountInfo = await connection.getAccountInfo(configCorePDA);

if (!accountInfo) throw new Error("ConfigCore not found");

const config = ConfigCoreSchema.decode(accountInfo.data);
```

---

## ConfigAssets

### PDA

```
["config", "assets"]
```

### Purpose

Maps asset name → asset PDA.

### Query Example

```ts
const [configAssetsPDA] = findConfigAssetsAddress(PROGRAM_ID);
const accountInfo = await connection.getAccountInfo(configAssetsPDA);

const config = ConfigAssetsSchema.decode(accountInfo.data);
```

Returned structure:

```
{
  is_initialized: bool,
  asset_mappings: [
    { name: string, address: PublicKey }
  ],
  bump: u8
}
```

---

## ConfigPairs

### PDA

```
["config", "pairs"]
```

### Purpose

Stores all registered trading pairs.

Example:

```
BTC-USD
ETH-USD
```

### Query Example

```ts
const [configPairsPDA] = findConfigPairsAddress(PROGRAM_ID);
const accountInfo = await connection.getAccountInfo(configPairsPDA);

const config = ConfigPairsSchema.decode(accountInfo.data);
```

---

## ConfigKeepers

### PDA

```
["config", "keepers"]
```

### Purpose

Authorized keeper public keys.

### Query Example

```ts
const [configKeepersPDA] = findConfigKeepersAddress(PROGRAM_ID);
const accountInfo = await connection.getAccountInfo(configKeepersPDA);

const config = ConfigKeepersSchema.decode(accountInfo.data);
```

---

# Trading Data Queries

Trading data is user-specific.

---

# UserAccount

### PDA

```
["user_account", user, tokenMint]
```

### Purpose

Tracks:

- Balances
- Orders
- Limit orders
- Funding history
- Trading stats

### Query Example

```ts
const [userAccountPDA] = findUserAccountAddress(
  PROGRAM_ID,
  userPubkey,
  tokenMint
);

const accountInfo = await connection.getAccountInfo(userAccountPDA);

const userAccount = UserAccountSchema.decode(accountInfo.data);
```

---

# Positions

### PDA

```
["position", user, pair, direction]
```

direction:
```
0 = LONG
1 = SHORT
```

### Query Logic

You must check **both directions** for each pair.

```ts
for (const direction of [0, 1]) {
  const [positionPDA] = findPositionAddress(
    PROGRAM_ID,
    userPubkey,
    pair,
    direction
  );

  const accountInfo = await connection.getAccountInfo(positionPDA);

  if (accountInfo) {
    const position = PositionSchema.decode(accountInfo.data);
  }
}
```

---

# Limit Orders

### PDA

```
["limit_order", orderId]
```

Order IDs are stored in:

```
userAccount.limit_orders
```

### Query Example

```ts
for (const orderId of userAccount.limit_orders) {
  const [pda] = findLimitOrderAddress(PROGRAM_ID, orderId);
  const accountInfo = await connection.getAccountInfo(pda);

  const order = LimitedOrderSchema.decode(accountInfo.data);
}
```

---

# Deals

### PDA

```
["deal", orderId]
```

Order IDs are stored in:

```
userAccount.orders
```

### Query Example

```ts
for (const dealId of userAccount.orders) {
  const [pda] = findDealAddress(PROGRAM_ID, dealId);
  const accountInfo = await connection.getAccountInfo(pda);

  const deal = DealSchema.decode(accountInfo.data);
}
```

---

# Query Execution Flow (User Trading Data)

1. Fetch UserAccount
2. Extract:
   - `orders`
   - `limit_orders`
3. Fetch:
   - Positions (LONG + SHORT per pair)
   - Limit Orders
   - Deals
4. Decode using matching Borsh schemas

---

# Important Notes

- All numeric values are `u128`, `u64`, or `i64` and should be handled as `bigint`.
- Always normalize BN → bigint after decoding.
- If decoding fails:
  - Check schema ordering
  - Check account size
  - Confirm correct PDA derivation
- Positions exist even if size = 0; check `size > 0` to filter active ones.

---

All queries are deterministic and require no program execution.
