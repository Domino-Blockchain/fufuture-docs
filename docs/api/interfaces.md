---
id: interfaces
title: Interfaces
sidebar_position: 5
---

# Fufuture API – Interface Collection

This file contains TypeScript interfaces only.  
No implementations.  
These match on-chain Borsh structures and instruction payloads.
Work in progress.

---

## ENUMS

```ts
import { PublicKey } from '@solana/web3.js'

export enum Direction {
  LONG = 0,
  SHORT = 1,
}

export enum OrderType {
  MARKET = 0,
  LIMIT = 1,
}

export enum Offset {
  OPEN = 1,
  CLOSE = 2,
}

export enum OrderState {
  PENDING = 0,
  PARTIAL = 1,
  COMPLETED = 2,
  CANCELLED = 3,
  EXPIRED = 4,
  LIQUIDATED = 5,
  EXCEPTION = 6,
}

export enum DealState {
  OPENING = 0,
  OPENED = 1,
  CLOSING = 2,
  CLOSED = 3,
  LIQUIDATING = 4,
  LIQUIDATED = 5,
}
````

---

## CONFIG STRUCTURES

```ts
export interface ConfigPairs {
  is_initialized: boolean
  trading_pairs: string[]
  bump: number
}

export interface ConfigKeepers {
  is_initialized: boolean
  keepers: PublicKey[]
  bump: number
}
```

---

## POSITION

```ts
export interface Position {
  is_initialized: boolean
  user: PublicKey
  pair: string
  direction: Direction
  value: bigint        // token decimals
  size: bigint         // E18
  margin: bigint       // token decimals
  freeze: bigint       // E18
  deals: bigint[]
  bump: number
}
```

---

## LIMIT ORDER

```ts
export interface LimitedOrder {
  is_initialized: boolean
  limit_order_id: bigint
  taker: PublicKey
  direct: Direction
  state: OrderState
  offset: Offset
  name: string
  amount: bigint       // E18
  target_price: bigint // E18
  margin: bigint       // token decimals
  trading_fee: bigint  // token decimals
  reward_gas: bigint   // lamports
  start_time: bigint
  good_till: bigint
  bump: number
}
```

---

## DEAL

```ts
export interface Deal {
  is_initialized: boolean
  order_id: bigint
  taker: PublicKey
  maker: PublicKey
  direct: Direction
  offset: Offset
  state: DealState
  name: string
  amount: bigint        // E18
  price: bigint         // E18
  margin: bigint        // token decimals
  trading_fee: bigint   // token decimals
  reward_gas: bigint
  open_time: bigint
  close_time: bigint
  close_price: bigint
  profit_loss: bigint   // signed i128
  bump: number
}
```

---

## USER ACCOUNT

```ts
export interface TokenAccountBalance {
  token: PublicKey
  available_amount: bigint
  locked_amount: bigint
  deposit_amount: bigint
  freeze: bigint
  last_funding_payment: bigint
  total_funding_paid: bigint
  total_trading_fees: bigint
}

export interface UserAccount {
  is_initialized: boolean
  user: PublicKey
  owner: PublicKey
  token: PublicKey
  available_amount: bigint
  locked_amount: bigint
  deposit_amount: bigint
  freeze: bigint
  last_funding_payment: bigint
  total_funding_paid: bigint
  total_trading_fees: bigint
  active_orders: number
  active_limit_orders: number
  created_at: bigint
  bump: number
  orders: bigint[]
  limit_orders: bigint[]
  order_ids: bigint[]
  limit_order_ids: bigint[]
  token_accounts: TokenAccountBalance[]
}
```

---

## POOL

```ts
export interface Pool {
  is_initialized: boolean
  token_mint: PublicKey
  underlying_name: string
  total_liquidity: bigint
  total_shares: bigint
  is_reject_order: boolean
  bump: number
}

export interface ShareHolder {
  provider: PublicKey
  shares: bigint
}
```

---

## INSTRUCTION PARAMS

```ts
export interface OpenLimitParams {
  pair: string
  taker: PublicKey
  amount: bigint
  price: bigint
  direction: Direction
  goodTill: bigint
  leverage: number
}

export interface CloseLimitParams {
  pair: string
  amount: bigint
  price: bigint
  direction: Direction
  goodTill: bigint
}

export interface TradeFuturesParams {
  pair: string
  amount: bigint
  price: bigint
  orderType: OrderType
  direction: Direction
  goodTill: bigint
  taker: PublicKey
  rewardGas: bigint
}

export interface ClosePositionParams {
  pair: string
  amount: bigint
  orderType: OrderType
  price: bigint
  direction: Direction
  goodTill: bigint
  rewardGas: bigint
  dealIds: bigint[]
}

export interface DepositParams {
  amount: bigint
}

export interface WithdrawParams {
  amount: bigint
  symbols: string[]
}

export interface AddLiquidityParams {
  amount: bigint
}

export interface WithdrawLiquidityParams {
  lpAmount: bigint
}
```

---

## EVENTS

```ts
export interface TradeHistoryEvent {
  type: 'TradeHistory'
  orderId: bigint
  amount: bigint
  price: bigint
  profit: bigint
  loss: bigint
}

export interface LiquidationEvent {
  type: 'Liquidation'
  orderId: bigint
  amount: bigint
  price: bigint
  penalty: bigint
}

export type ProgramEvent =
  | TradeHistoryEvent
  | LiquidationEvent
```

---
