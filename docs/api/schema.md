---
id: schemas
title: Borsh Schemas
sidebar_position: 3
---

# Borsh Schemas

This section defines the complete Borsh serialization schemas used for all read-only query methods.

All schemas must match the Rust struct layout exactly (field order, types, and integer widths).

---

## 1. ConfigPairs Schema

### Rust Struct (config.rs)

```
pub struct ConfigPairs {
    pub is_initialized: bool,
    pub trading_pairs: Vec<String>,
    pub bump: u8,
}
```

### Borsh Schema

```ts
import * as borsh from '@coral-xyz/borsh';

export const ConfigPairsSchema = borsh.struct([
  borsh.bool('is_initialized'),
  borsh.vec(borsh.str(), 'trading_pairs'),
  borsh.u8('bump'),
]);
```

---

## 2. Position Schema

### Rust Struct (position.rs)

```
pub struct Position {
    pub is_initialized: bool,
    pub user: Pubkey,
    pub pair: String,
    pub direction: u8,
    pub value: u128,
    pub size: u128,
    pub margin: u128,
    pub freeze: u128,
    pub deals: Vec<u64>,
    pub bump: u8,
}
```

### Borsh Schema

```ts
export const PositionSchema = borsh.struct([
  borsh.bool('is_initialized'),
  borsh.publicKey('user'),
  borsh.str('pair'),
  borsh.u8('direction'),
  borsh.u128('value'),
  borsh.u128('size'),
  borsh.u128('margin'),
  borsh.u128('freeze'),
  borsh.vec(borsh.u64(), 'deals'),
  borsh.u8('bump'),
]);
```

---

## 3. LimitedOrder Schema

### Enums

```ts
export const OrderStateEnum = borsh.rustEnum([
  borsh.struct([], 'Pending'),
  borsh.struct([], 'Partial'),
  borsh.struct([], 'Completed'),
  borsh.struct([], 'Cancelled'),
  borsh.struct([], 'Expired'),
  borsh.struct([], 'Liquidated'),
  borsh.struct([], 'Exception'),
]);

export const DirectEnum = borsh.rustEnum([
  borsh.struct([], 'Long'),
  borsh.struct([], 'Short'),
]);

export const OffsetEnum = borsh.rustEnum([
  borsh.struct([], 'Open'),
  borsh.struct([], 'Close'),
]);
```

### Rust Struct (limit_order.rs)

```
pub struct LimitedOrder {
    pub is_initialized: bool,
    pub limit_order_id: u64,
    pub taker: Pubkey,
    pub direct: Direct,
    pub state: OrderState,
    pub offset: Offset,
    pub name: String,
    pub amount: u128,
    pub target_price: u128,
    pub margin: u128,
    pub trading_fee: u128,
    pub reward_gas: u64,
    pub start_time: i64,
    pub good_till: i64,
    pub bump: u8,
}
```

### Borsh Schema

```ts
export const LimitedOrderSchema = borsh.struct([
  borsh.bool('is_initialized'),
  borsh.u64('limit_order_id'),
  borsh.publicKey('taker'),
  DirectEnum.replicate('direct'),
  OrderStateEnum.replicate('state'),
  OffsetEnum.replicate('offset'),
  borsh.str('name'),
  borsh.u128('amount'),
  borsh.u128('target_price'),
  borsh.u128('margin'),
  borsh.u128('trading_fee'),
  borsh.u64('reward_gas'),
  borsh.i64('start_time'),
  borsh.i64('good_till'),
  borsh.u8('bump'),
]);
```

---

## 4. Deal Schema

### DealState Enum

```ts
export const DealStateEnum = borsh.rustEnum([
  borsh.struct([], 'Opening'),
  borsh.struct([], 'Opened'),
  borsh.struct([], 'Closing'),
  borsh.struct([], 'Closed'),
  borsh.struct([], 'Liquidating'),
  borsh.struct([], 'Liquidated'),
]);
```

### Rust Struct (deal.rs)

```
pub struct Deal {
    pub is_initialized: bool,
    pub order_id: u64,
    pub taker: Pubkey,
    pub maker: Pubkey,
    pub direct: Direct,
    pub offset: Offset,
    pub state: DealState,
    pub name: String,
    pub amount: u128,
    pub price: u128,
    pub margin: u128,
    pub trading_fee: u128,
    pub reward_gas: u64,
    pub open_time: i64,
    pub close_time: i64,
    pub close_price: u128,
    pub profit_loss: i128,
    pub bump: u8,
}
```

### Borsh Schema

```ts
export const DealSchema = borsh.struct([
  borsh.bool('is_initialized'),
  borsh.u64('order_id'),
  borsh.publicKey('taker'),
  borsh.publicKey('maker'),
  DirectEnum.replicate('direct'),
  OffsetEnum.replicate('offset'),
  DealStateEnum.replicate('state'),
  borsh.str('name'),
  borsh.u128('amount'),
  borsh.u128('price'),
  borsh.u128('margin'),
  borsh.u128('trading_fee'),
  borsh.u64('reward_gas'),
  borsh.i64('open_time'),
  borsh.i64('close_time'),
  borsh.u128('close_price'),
  borsh.i128('profit_loss'),
  borsh.u8('bump'),
]);
```

---

## 5. UserAccount Schema

### Nested TokenAccount Struct

```ts
export const TokenAccountSchema = borsh.struct([
  borsh.publicKey('token'),
  borsh.u128('available_amount'),
  borsh.u128('locked_amount'),
  borsh.u128('deposit_amount'),
  borsh.u128('freeze'),
  borsh.i64('last_funding_payment'),
  borsh.u128('total_funding_paid'),
  borsh.u128('total_trading_fees'),
]);
```

### Rust Struct (user_account.rs)

```
pub struct UserAccount {
    pub is_initialized: bool,
    pub user: Pubkey,
    pub owner: Pubkey,
    pub token: Pubkey,
    pub available_amount: u128,
    pub locked_amount: u128,
    pub deposit_amount: u128,
    pub freeze: u128,
    pub last_funding_payment: i64,
    pub total_funding_paid: u128,
    pub total_trading_fees: u128,
    pub active_orders: u32,
    pub active_limit_orders: u32,
    pub created_at: i64,
    pub bump: u8,
    pub orders: Vec<u64>,
    pub limit_orders: Vec<u64>,
    pub order_ids: Vec<u64>,
    pub limit_order_ids: Vec<u64>,
    pub token_accounts: Vec<TokenAccount>,
}
```

### Borsh Schema

```ts
export const UserAccountSchema = borsh.struct([
  borsh.bool('is_initialized'),
  borsh.publicKey('user'),
  borsh.publicKey('owner'),
  borsh.publicKey('token'),
  borsh.u128('available_amount'),
  borsh.u128('locked_amount'),
  borsh.u128('deposit_amount'),
  borsh.u128('freeze'),
  borsh.i64('last_funding_payment'),
  borsh.u128('total_funding_paid'),
  borsh.u128('total_trading_fees'),
  borsh.u32('active_orders'),
  borsh.u32('active_limit_orders'),
  borsh.i64('created_at'),
  borsh.u8('bump'),
  borsh.vec(borsh.u64(), 'orders'),
  borsh.vec(borsh.u64(), 'limit_orders'),
  borsh.vec(borsh.u64(), 'order_ids'),
  borsh.vec(borsh.u64(), 'limit_order_ids'),
  borsh.vec(TokenAccountSchema, 'token_accounts'),
]);
```

---

All frontend deserialization must use these schemas exactly as defined above to match on-chain layout.
