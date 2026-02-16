---
id: orderbook-state
title: Orderbook State
sidebar_position: 11
---

# Orderbook State

The `Orderbook State` account tracks global order execution state for a pool.

It stores:

- Core metadata
- Pool references
- Matching configuration
- `order_id` counter (u64)

This counter increments every time a new `TradeFutures` instruction executes.

---

# Account Creation

The orderbook is a **program-owned account**, not a PDA.

It is created using:

```ts
SystemProgram.createAccount({
  fromPubkey: authority.publicKey,
  newAccountPubkey: orderbookKeypair.publicKey,
  space: ORDERBOOK_STATE_LEN,
  lamports,
  programId: PROGRAM_ID,
});
```

---

# Account Size

```ts
const ORDERBOOK_STATE_LEN = 1 + (32 * 5) + 8; // 169 bytes
```

Layout:

| Field | Size |
|-------|------|
| is_initialized | 1 |
| 5 Pubkeys | 160 |
| order_id (u64) | 8 |

---

# Reading Current Order ID

```ts
const orderbookData = await connection.getAccountInfo(orderbookKeypair.publicKey);

const orderIdOffset = 1 + (32 * 5);
const currentOrderId = orderbookData.data.readBigUInt64LE(orderIdOffset);
```

This `currentOrderId` must be used when deriving:

- Deal PDA
- Trade execution

---

# Lifecycle Role

The orderbook:

- Tracks sequential deal IDs
- Prevents duplicate deal creation
- Ensures deterministic PDA derivation
- Coordinates matching state

Without it, `TradeFutures` cannot execute.

---