---
id: v2-flows
title: Legacy ↔ V2 Flow Comparison & Parameter Manifest
sidebar_position: 6
---

# Legacy ↔ V2 Flow Comparison & Parameter Manifest

This document maps each legacy transaction flow to its V2 parity-lane equivalent and
provides a parameter manifest template for integrators. V2 facts are anchored to
`src/v2/`; legacy facts to the existing [API Reference](../api/instructions.md).

> **Compatibility reminder:** both lanes share one Program ID; legacy tags 0–59 are
> unchanged; account layouts are **not** interchangeable. Oracle reads are identical
> (Custos/PPRC, E6 → ×1000 → E9, 150 slots / 200 bps).

---

## 1. Trading: single-tx fill → two-phase order

**Legacy (`TradeFutures`, tag 7).** One instruction submitted by the keeper creates
the `Deal` PDA, creates/updates the `Position`, and updates pool/user state
atomically at submission time (see [tradeFutures](../api/trading/tradeFutures.md)).

**V2.** Two phases, mirroring the Solana futures-v2 model:

1. **Make** (user-signed): `MakeOpenOrderV2` (93) / `MakeCloseOrderV2` (94) /
   `MakeStopTakeOrderV2` (95). The client reads `sequence.next_order_seq` and passes
   it as `order_seq` (`V2OrderSeqMismatch` 431 on race). The program creates the
   `order` + `trigger` PDAs, escrows margin+fee in `trader.order_locked_base` (opens)
   or freezes position size (closes), and escrows `reward_gas` lamports in the order
   PDA. Events: `OrderCreated` + `TriggerConditionCreated`.
2. **Trigger** (keeper-signed): `TriggerOpenOrderV2` (96) / `TriggerCloseOrderV2`
   (97) / `TriggerStopTakeOpenV2` (98) / `TriggerStopTakeCloseV2` (99). The keeper
   re-validates the price condition against the Custos/PPRC mark, creates the deal
   PDA (`deal_id = 0xC000… | order_seq`), accrues fees **synchronously**, pays the
   escrowed reward to itself, and returns order/trigger rent to the taker.
   Events: `DealOpened` + `FeeAccrued` + `OrderTerminated state=3`.

A stop-take order may convert instead of filling (`OrderConverted`): stop-take open →
limit open (limit branch), stop-take open → attached stop-take close (market fill
with TP/SL legs), stop-take close → limit close. Each conversion pays half the
escrowed reward to the keeper and keeps the rest for the converted order.

**Close parity note:** a V2 close settles **a single Deal** (`deal_id` arg on tags
97/99) with pro-rata margin release — there is no legacy-style multi-deal sweep in
one trigger.

## 2. Conditional orders: batch trigger → per-order trigger

**Legacy (`TriggerOrders`, tag 12).** One keeper instruction executes a batch of
triggered conditional orders.

**V2.** Each order is triggered individually (tags 96–99) because every fill creates
its own deal PDA and fee accrual. Batch capacity is bounded by accounts-per-tx; the
keeper iterates pending orders off-chain. `ExpireOrderV2` (101) and
`CleanupOrphanOrdersV2` (102) replace legacy orderbook GC: expire rewards the keeper
from the escrowed `reward_gas`; orphan cleanup targets close orders whose position is
already empty (`reason=orphan`).

## 3. Maker liquidation: two transactions → one packet

**Legacy.** Two-step: `LiquidateMakerDeals` (tag 17) marks/debits maker deals, then
`UpdateTakerBalances` (tag 42) credits takers in a second transaction.

**V2.** `LiquidateMakerDealsBulkV2` (105) settles **1–4 deals atomically** in one
instruction: per deal it either auto-adds maker margin (`MakerMarginAdded
source=auto`) or force-closes (`DealAgreement`), crediting the taker in the same
transaction (margin + capped `credited_gain`; maker shortfall → risk fund → bad
debt). No follow-up instruction is required.

## 4. Taker liquidation: immediate → atomic packets with nonce + receipt

**Legacy.** Liquidation executes immediately against account state with no
concurrency guard.

**V2.** `LiquidateAccountBulkV2` (103) is an **atomic, optimistic-concurrency
packet**:

- The client snapshots `trader.liquidation_nonce` and passes
  `expected_margin_nonce = nonce`, `receipt_nonce = nonce + 1`; staleness →
  `V2StaleNonce` (405).
- Up to 32 deals per packet (`V2BulkDealCapExceeded` 455), grouped by liquidity page
  via `page_groups` (strictly increasing, counts must sum).
- Each packet writes a `LiquidationReceiptV2` PDA (`["liq-receipt", mint, trader,
  receipt_nonce]`) — replay protection via nonce + closed flag (`V2ReceiptReplay`
  406), plus a SHA-256 `outcomes_hash` audit event.
- Multi-packet sessions: first packet requires the account to be at risk
  (`V2NotLiquidatable` 454) and flips status ACTIVE → LIQUIDATING; continuation
  packets never re-check. Final packet sweeps residual equity to the risk fund and
  flips to LIQUIDATED.
- After LIQUIDATED, `LiquidateCancelOrdersV2` (104) closes the trader's remaining
  pending orders (reward → keeper, rent → taker, `reason=liquidation`).
- Liquidation closes charge **no trading fee** (`trading_fee_base=0` literal in
  `DealLiquidated`); normal closes still charge.

## 5. Fees: deferred distribution → synchronous accrual + self-service claim

**Legacy (`DistributeFees`, tag 52).** Fees are credited later by a separate
distribution instruction; inviter rewards are only credited as claimable balances.

**V2.** Every fill accrues fees **in the same transaction** via the fixed 7-account
fee bundle (`accrue_fill_fee_v2`, `src/v2/fee.rs:75`): the total is split by the
settlement fee rates (`trading_fee_to_platform_e9` default 0.2e9,
`trading_fee_to_inviter_e9` 0.1e9, `trading_fee_to_top_agent_e9` 0.5e9) into
`settlement.platform_fee_accrued_base`, `trade_fee_accrued_base`, and the inviter /
approved top-agent `CommissionAccountV2`s. Unattributed shares fold into the trade
fee. A conservation check (`V2ConservationFailure` 409) ends every accrual.

Claims are self-service and can be called any time the protocol is not paused:

- `ClaimPlatformFeesV2` (72) / `ClaimTradeFeesV2` (73) — signed by the configured
  recipient owner; zeroes the accrued counter and pays out from custody.
- `ClaimCommissionV2` (74) — signed by the commission account owner;
  pays `accrued − claimed`.

## 6. Liquidity: legacy pool → public shares / private maker slots

- Public pool: configured once by `CreatePublicPoolV2` (76) on page 0; depositors
  hold `PublicShareV2` PDAs and enter/exit pro-rata to effective equity
  (`ProvideLiquidityV2`/`WithdrawLiquidityV2` with `pool_kind = 1`), with oracle
  marks and `limit_out` slippage protection.
- Private pools: `CreatePrivatePoolV2` (77) allocates a maker slot on any page;
  makers tune risk with `SetMakerParamsV2` (82), top up deals with
  `AddMakerMarginV2` (84) / `AddMakerMarginFromPoolV2` (85); keepers freeze
  underwater makers with `LiquidateLpV2` (86); admins repair public-slot
  contamination with `RepairPublicSlotPrivateLiquidityV2` (87).

---

## 7. Parameter Manifest Template

Use this template when configuring or integrating a V2 settlement. All E9 values are
1e9 fixed-point; `_base` values are settlement-mint base units; rewards are native
lamports.

```yaml
program:
  program_id: <same as legacy deployment>      # one Program ID for both lanes
  instruction_tags: 60..107                    # V2 lane; legacy 0..59 unchanged

oracle:
  reader: Custos/PPRC PriceUpdateV2            # src/oracle/pyth.rs
  policy: ORACLE_POLICY_DOMI_CUSTOS_V1 (=1)
  max_staleness_slots: 150
  max_confidence_bps: 200
  precision: raw pyth → E6 (raw_price_to_e6) → ×1000 → E9 (oracle_e6_to_e9)

protocol:                                      # tag 60 / 61 / 75 / 88 / 92
  admin: <pubkey>
  maintenance_margin_rate_e9: <e9>             # (0, 1e9]
  min_deposit_amount: <base>
  order_switch: 0b111                          # market|limit|stop-take; stored 0 = all on
  permissionless_markets: true                 # reserved[1]
  keepers: [<pubkey>, ...]                     # ≤ 16

settlement:                                    # created by tag 76/77; tuned by tag 62
  mint: <pubkey>
  maintenance_margin_rate_e9: 0                # 0 = inherit protocol MMR
  min_deposit_amount: <base>                   # overrides protocol when > 0
  fee_split_e9:
    platform: 200_000_000                      # 0.2
    inviter: 100_000_000                       # 0.1
    top_agent: 500_000_000                     # 0.5 (sum ≤ 1e9)
  lifecycle_flags: 0b111                       # deal-id-v2 | deal-reclaim | receipt-ephemeral-close

market_order_config:                           # tag 91, per pair 1..3
  default_leverage_e9: 100_000_000_000         # 100× (E9)
  min_order_amount_e9: 1
  max_market_open_deviation_e9: 50_000_000     # 5%
  minimum_reward_gas: 3_000_000                # lamports
  order_override_mask: 0
  order_override_value: 0

order_defaults:
  good_till_secs: 604800                       # 7 days when good_till <= 0
  tpsl_good_till_secs: 315360000               # ~10 years, set at stop-take conversion
  fast_lane_pairs: [1, 2, 3]

capacities:
  pending_orders_per_trader: 32
  position_slots_per_trader: 8
  deals_per_position: 50
  makers_per_page: 16
  deal_slots_per_page: 48
  taker_liquidation_packet: 32
  maker_liquidation_packet: 4
  receipt_outcomes: 32
  keepers: 16
  markets_per_page: 16

client_checklist:
  - Read sequence.next_order_seq; pass as order_seq (else V2OrderSeqMismatch 431)
  - reward_gas >= market_order_config.minimum_reward_gas (else 434)
  - open: amount_e9 >= min_order_amount_e9 (else 433); funds margin+fee escrowed
  - close: amount_e9 <= position.size_e9 - freeze_e9 (else 440)
  - trigger tx must append oracle price + 7-account fee bundle in fixed order
  - liquidation tx must pass liquidation_nonce / nonce+1 receipt (else 405/406)
  - price accounts: one per unique active pair, first-occurrence order (else 404)
```

---

## 8. Instruction Index (V2)

| Tag | Instruction | Signer | Pause-gated |
| --- | --- | --- | --- |
| 60 | InitializeProtocolV2 | payer | no |
| 61 | SetProtocolMaintenanceMarginRateV2 | admin | no |
| 62 | SetSettlementMaintenanceMarginRateV2 | admin | no |
| 63 | SetUserLeverageV2 | owner | no |
| 64 | InitializeTraderMarginV2 | owner | no |
| 65 | DepositMarginV2 | owner | protocol + settlement |
| 66 | WithdrawMarginV2 | owner | protocol only |
| 67 | InitializeFeeRecipientsV2 | admin | no |
| 68 | SetFeeRecipientsV2 | admin | no |
| 69 | InitializeCommissionAccountV2 | payer | no |
| 70 | BindInviterV2 | invitee | no |
| 71 | SetTopAgentV2 | admin | no |
| 72 | ClaimPlatformFeesV2 | recipient owner | protocol only |
| 73 | ClaimTradeFeesV2 | recipient owner | protocol only |
| 74 | ClaimCommissionV2 | commission owner | protocol only |
| 75 | SetPermissionlessMarketsV2 | admin | no |
| 76 | CreatePublicPoolV2 | payer (permissionless*) | no |
| 77 | CreatePrivatePoolV2 | holder (permissionless*) | no |
| 78 | InitializeLiquidityPageV2 | payer | no |
| 79 | InitializePublicShareV2 | holder | no |
| 80 | ProvideLiquidityV2 | holder | protocol + settlement |
| 81 | WithdrawLiquidityV2 | holder | protocol only |
| 82 | SetMakerParamsV2 | maker holder | no |
| 83 | SetEscrowParamsV2 | admin | no |
| 84 | AddMakerMarginV2 | maker holder | no |
| 85 | AddMakerMarginFromPoolV2 | maker holder | no |
| 86 | LiquidateLpV2 | keeper | no |
| 87 | RepairPublicSlotPrivateLiquidityV2 | admin | no |
| 88 | SetKeeperV2 | admin | no |
| 89 | UpsertMarketTemplateV2 | admin | no |
| 90 | BootstrapMarketsV2 | payer (permissionless*) | no |
| 91 | SetMarketOrderConfigV2 | admin | no |
| 92 | SetOrderSwitchV2 | admin | no |
| 93 | MakeOpenOrderV2 | taker | protocol + settlement + market |
| 94 | MakeCloseOrderV2 | taker | protocol + settlement (+ market OFFLINE) |
| 95 | MakeStopTakeOrderV2 | taker | protocol + settlement + market |
| 96 | TriggerOpenOrderV2 | keeper | protocol + settlement |
| 97 | TriggerCloseOrderV2 | keeper | **no** |
| 98 | TriggerStopTakeOpenV2 | keeper | protocol + settlement |
| 99 | TriggerStopTakeCloseV2 | keeper | **no** |
| 100 | CancelOrderV2 | taker | **no** |
| 101 | ExpireOrderV2 | keeper | no |
| 102 | CleanupOrphanOrdersV2 | keeper | no |
| 103 | LiquidateAccountBulkV2 | keeper | **no** |
| 104 | LiquidateCancelOrdersV2 | keeper | **no** |
| 105 | LiquidateMakerDealsBulkV2 | keeper | **no** |
| 106 | DepositRiskFundV2 | admin | no |
| 107 | WithdrawRiskFundV2 | admin | no |

\* permissionless = allowed for anyone once `protocol.permissionless_markets()` is
enabled (`V2PermissionlessDisabled` 420 otherwise).
