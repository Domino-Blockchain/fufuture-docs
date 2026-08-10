---
id: v2-events
title: V2 Event Reference
sidebar_position: 4
---

# V2 Event Reference

V2 instructions emit **text events** via `msg!` log lines. Every event line starts
with the literal prefix `DOMI_V2_EVENT ` followed by space-separated `key=value`
pairs. All formats below are copied verbatim from `src/v2/` (`{}` = substituted
placeholder); anchors point at the `msg!` call.

## Envelope

Common leading fields (present unless noted):

- `protocol_version` — always `1` (`CURRENT_VERSION`).
- `slot` — Domichain slot. **Exception:** the `terminate_order` variant of
  `OrderTerminated` omits `slot` (see below).
- `timestamp` — unix seconds.
- `event_index` — index of the event within the instruction (0-based). Parse per
  transaction, not per log line.
- `settlement` — always the settlement **mint** pubkey, never the settlement PDA.

Units are encoded in field names: `_e9` (1e9 scale), `_base` (settlement base units),
`reward_gas`/`lamports` (native lamports). No event carries a `unit=` field except
`MaintenanceMarginRateUpdated` and `LeverageSet` (`unit=e9`).

---

## Protocol & Config Events

```
event=ProtocolInitialized protocol_version={} slot={} timestamp={} event_index=0 admin={} maintenance_margin_rate_e9={} min_deposit_amount_base={}
```
`src/v2/processor.rs:129` — tag 60 InitializeProtocolV2.

```
event=MaintenanceMarginRateUpdated protocol_version={} slot={} timestamp={} event_index=0 scope=protocol unit=e9 old_value={} new_value={}
event=MaintenanceMarginRateUpdated protocol_version={} slot={} timestamp={} event_index=0 scope=settlement unit=e9 settlement={} old_value={} new_value={}
```
`src/v2/processor.rs:158` (tag 61) and `:195` (tag 62). Note settlement MMR accepts
`new_value=0` = "inherit protocol default".

```
event=LeverageSet protocol_version={} slot={} timestamp={} event_index=0 owner={} settlement={} pair_id={} unit=e9 old_value={} new_value={}
```
`src/v2/processor.rs:295` — tag 63. `old_value=0` on first creation.

```
event=PermissionlessMarketsUpdated protocol_version={} slot={} timestamp={} event_index=0 enabled={}
```
`src/v2/pool.rs:394` — tag 75.

```
event=KeeperUpdated protocol_version={} slot={} timestamp={} event_index=0 keeper={} enabled={}
```
`src/v2/pool.rs:417` — tag 88.

```
event=MarketTemplateUpserted protocol_version={} slot={} timestamp={} event_index=0 pair_id={} name={}
```
`src/v2/market.rs:93` — tag 89.

```
event=MarketsBootstrapped protocol_version={} slot={} timestamp={} event_index=0 payer={} settlement={} market_count={} pair_min={} pair_max={} default_leverage_e9={} min_order_amount_e9={} max_market_open_deviation_e9={} minimum_reward_gas={}
```
`src/v2/market.rs:400` — tag 90.

```
event=MarketOrderConfigUpdated protocol_version={} slot={} timestamp={} event_index=0 settlement={} pair_id={} old_default_leverage_e9={} new_default_leverage_e9={} old_min_order_amount_e9={} new_min_order_amount_e9={} old_max_market_open_deviation_e9={} new_max_market_open_deviation_e9={} old_minimum_reward_gas={} new_minimum_reward_gas={}
```
`src/v2/market.rs:465` — tag 91.

```
event=OrderSwitchUpdated protocol_version={} slot={} timestamp={} event_index=0 old_value={} new_value={}
```
`src/v2/market.rs:504` — tag 92. Values are the *effective* switch (0 stored = `0b111`).

---

## Funds Events

```
event=TraderMarginInitialized protocol_version={} slot={} timestamp={} event_index=0 owner={} settlement={}
```
`src/v2/funds.rs:322` — tag 64.

```
event=MarginDeposited protocol_version={} slot={} timestamp={} event_index=0 owner={} settlement={} amount_base={} equity_after_base={}
```
`src/v2/funds.rs:409` — tag 65.

```
event=MarginWithdrawn protocol_version={} slot={} timestamp={} event_index=0 owner={} settlement={} amount_base={} net_loss_reserved_base={} equity_after_base={}
```
`src/v2/funds.rs:516` — tag 66. `net_loss_reserved_base` = net unrealized loss the
risk gate reserved.

## Fee & Referral Events

```
event=FeeRecipientsInitialized protocol_version={} slot={} timestamp={} event_index=0 platform_owner={} trade_fee_owner={}
event=FeeRecipientsUpdated protocol_version={} slot={} timestamp={} event_index=0 admin={} old_platform_owner={} new_platform_owner={} old_trade_fee_owner={} new_trade_fee_owner={}
```
`src/v2/fee.rs:263` (tag 67), `:293` (tag 68).

```
event=InviterBound protocol_version={} slot={} timestamp={} event_index=0 invitee={} inviter={}
```
`src/v2/fee.rs:395` — tag 70. (Tag 69 InitializeCommissionAccountV2 emits **no** event.)

```
event=TopAgentUpdated protocol_version={} slot={} timestamp={} event_index=0 settlement={} agent={} approved={}
```
`src/v2/fee.rs:465` — tag 71.

```
event=ProtocolFeeClaimed protocol_version={} slot={} timestamp={} event_index=0 settlement={} fee_kind={} owner={} receiver_token={} amount_base={}
```
`src/v2/fee.rs:545` — tags 72/73. `fee_kind`: 1 = platform, 2 = trade.

```
event=CommissionClaimed protocol_version={} slot={} timestamp={} event_index=0 settlement={} owner={} receiver_token={} amount_base={}
```
`src/v2/fee.rs:624` — tag 74.

---

## Liquidity Events

```
event=SettlementInitialized protocol_version={} slot={} timestamp={} event_index=0 settlement={} custody={} decimals={}
```
`src/v2/pool.rs:487` — emitted by tags 76/77 when the settlement PDA is created
(first pool creation for a mint).

```
event=PublicPoolConfigured protocol_version={} slot={} timestamp={} event_index={} settlement={} page_id=0 maker_slot={}
```
`src/v2/pool.rs:498` — tag 76, first call only. `event_index` is 1 if the settlement
was created in the same instruction, else 0.

```
event=PrivateLiquidityProvided protocol_version={} slot={} timestamp={} event_index={} holder={} settlement={} page_id={} maker_slot={} amount_base={} equity_after_base={} available_after_base={}
```
`src/v2/pool.rs:597` (tag 77, `event_index` 0/1 as above) and `:791` (tag 80 private
path, always `event_index=0`).

```
event=LiquidityPageInitialized protocol_version={} slot={} timestamp={} event_index=0 settlement={} page_id={} page_count={}
```
`src/v2/pool.rs:657` — tag 78.

```
event=PublicShareInitialized protocol_version={} slot={} timestamp={} event_index=0 holder={} settlement={}
```
`src/v2/pool.rs:707` — tag 79.

```
event=PublicShareMinted protocol_version={} slot={} timestamp={} event_index=0 holder={} settlement={} amount_base={} shares={} effective_equity_before_base={} total_shares_after={}
```
`src/v2/pool.rs:869` — tag 80 public path.

```
event=LiquidityWithdrawn protocol_version={} slot={} timestamp={} event_index=0 holder={} settlement={} pool_kind={} page_id={} maker_slot={} input_amount={} payout_base={}
```
`src/v2/pool.rs:1055` — tag 81. `pool_kind`: 1 = public, 2 = private. `input_amount`
is base units (private) or shares (public); `payout_base` is the token amount paid out.

```
event=MakerParamsUpdated protocol_version={} slot={} timestamp={} event_index=0 authority={} settlement={} page_id={} maker_slot={} leverage_e9={} mmr_e9={} add_margin_rate_e9={} auto_add_margin={} reject_order={}
```
`src/v2/pool.rs:1132` (tag 82, `authority` = maker holder) and `:1190` (tag 83,
`authority` = admin, always targets the public slot). All five values are the
post-update maker fields.

```
event=MakerMarginAdded protocol_version={} slot={} timestamp={} event_index=0 source=trader maker={} settlement={} page_id={} maker_slot={} deal_id={} amount_base={} margin_after_base={} locked_after_base={}
event=MakerMarginAdded protocol_version={} slot={} timestamp={} event_index=0 source=pool maker={} settlement={} page_id={} maker_slot={} deal_id={} amount_base={} margin_after_base={} available_after_base={}
event=MakerMarginAdded protocol_version={} slot={} timestamp={} event_index={} source=auto maker={} settlement={} page_id={} maker_slot={} deal_id={} amount_base={} margin_after_base={} available_after_base={}
```
Tag 84 (`src/v2/pool.rs:1278`, literal `source=trader`, ends with `locked_after_base`),
tag 85 (`src/v2/pool.rs:1347`, `source=pool`, ends with `available_after_base`),
and the auto-add path of maker liquidation (`src/v2/liquidation.rs:1511`,
`source=auto`). Distinguish by the literal `source=` value.

```
event=LpFrozen protocol_version={} slot={} timestamp={} event_index=0 keeper={} holder={} settlement={} page_id={} maker_slot={} equity_base={} locked_base={}
```
`src/v2/pool.rs:1404` — tag 86. The maker is only frozen (reject_order + FROZEN flag);
funds move later through maker liquidation.

```
event=PublicSlotPrivateLiquidityRepaired protocol_version={} slot={} timestamp={} event_index=0 settlement={} holder={} public_page_id={} public_maker_slot={} private_maker_slot={} equity_base={} available_base={}
```
`src/v2/pool.rs:1455` — tag 87.

---

## Order Lifecycle Events

```
event=OrderCreated protocol_version={} slot={} timestamp={} event_index=0 order_seq={} taker={} settlement={} pair_id={} direction={} offset={} order_kind={} state={} amount_e9={} target_price_e9={} margin_base={} trading_fee_base={} reward_gas={} good_till={}
event=TriggerConditionCreated protocol_version={} slot={} timestamp={} event_index=1 order_seq={} mean={} open_limit_price_e9={} gain_trigger_price_e9={} gain_limit_price_e9={} loss_trigger_price_e9={} loss_limit_price_e9={}
```
`src/v2/order.rs:670` and `:690` — emitted together by tags 93/94/95
(`emit_order_created`). `direction`: 1 long / 2 short; `offset`: 1 open / 2 close;
`order_kind`: 1 limit-open, 2 limit-close, 3 stop-take-open, 4 stop-take-close,
5 market-open, 6 market-close; `state`: 1 (pending) at creation; `mean`: 0 for
limit/market orders, 1 (pure-condition) for stop-take.

```
event=DealOpened protocol_version={} slot={} timestamp={} event_index=0 history_seq={} order_seq={} deal_id={} taker={} settlement={} pair_id={} direction={} amount_e9={} price_e9={} margin_base={} trading_fee_base={} page_id={} maker_slot={} liquidity_deal_slot={} pool_kind={}
event=FeeAccrued protocol_version={} slot={} timestamp={} event_index=1 order_seq={} fee_total_base={} platform_base={} trade_fee_base={} direct_inviter_base={} top_agent_base={}
```
`src/v2/order.rs:1948` and `:1969` — open fills (tags 96/98). `deal_id` is the
trigger-derived id `0xC000… | order_seq`; `pool_kind`: 1 public / 2 private.

```
event=DealClosed protocol_version={} slot={} timestamp={} event_index=0 history_seq={} order_seq={} deal_id={} taker={} settlement={} pair_id={} direction={} amount_e9={} price_e9={} gain_base={} credited_gain_base={} loss_base={} margin_released_base={} trading_fee_base={} fully_closed={}
```
`src/v2/order.rs:2181` — close fills (tags 97/99). The close-fee accrual itself emits
no separate `FeeAccrued` line here.

### OrderTerminated — four variants (same `event=` name!)

Always key off the full field set, not just the event name.

```
event=OrderTerminated protocol_version={} timestamp={} event_index=0 order_seq={} taker={} settlement={} state={} reward_gas={} reward_receiver={} rent_receiver={}
```
Variant A/B, `terminate_order` / `finish_filled_order` (`src/v2/order.rs:1180`,
`:1924`) — **no `slot=` field, no `reason=`**.
- state=3 (completed): fill paths, reward → keeper, rent → taker.
- state=4 (cancelled): user cancel (tag 100, reward+rent → taker),
  reconciliation failure at fill, stop-take conversions with insufficient funds.
- state=6 (expired): keeper expire (tag 101, reward → keeper, rent → taker).

```
event=OrderTerminated protocol_version={} slot={} timestamp={} event_index=0 order_seq={} taker={} settlement={} state={} reason=orphan reward_gas={} reward_receiver={} rent_receiver={}
```
Variant C, orphan cleanup (`src/v2/order.rs:1323`, tag 102) — `reason=orphan`,
state=4, reward → keeper, rent → taker.

```
event=OrderTerminated protocol_version={} slot={} timestamp={} event_index={} order_seq={} taker={} settlement={} state={} reason=liquidation reward_gas={} reward_receiver={} rent_receiver={}
```
Variant D, liquidation cancel (`src/v2/liquidation.rs:1058`, tag 104) —
`reason=liquidation`, state=5 (liquidated), reward → keeper, rent → taker.

### OrderConverted — three shapes

```
event=OrderConverted protocol_version={} slot={} timestamp={} event_index=0 order_seq={} from_kind={} to_kind={} target_price_e9={} reward_paid={} reward_remaining={}
event=OrderConverted protocol_version={} slot={} timestamp={} event_index=0 order_seq={} from_kind={} to_kind={} target_price_e9={} margin_base={} trading_fee_base={} reward_paid={} reward_remaining={}
event=OrderConverted protocol_version={} slot={} timestamp={} event_index=2 order_seq={} from_kind={} to_kind={} target_price_e9={} reward_paid={} reward_remaining={}
```
- (a) `src/v2/order.rs:2365` — stop-take close (4) → limit close (2), tag 99.
- (b) `src/v2/order.rs:2629` — stop-take open (3) → limit open (1), tag 98;
  includes recomputed `margin_base`/`trading_fee_base`.
- (c) `src/v2/order.rs:2717` — stop-take open (3) → attached stop-take close (4)
  after the market fill, tag 98, `event_index=2`.

`reward_paid` is half the escrowed `reward_gas` paid to the keeper at conversion;
`reward_remaining` stays escrowed for the converted order.

---

## Liquidation Events

```
event=DealLiquidated protocol_version={} slot={} timestamp={} event_index={} history_seq={} order_seq=0 deal_id={} taker={} settlement={} pair_id={} direction={} amount_e9={} price_e9={} cost_price_e9={} leverage_e9={} margin_base={} gain_base={} loss_base={} bad_debt_base={} trading_fee_base=0 maker={} page_id={} maker_slot={} pool_kind={} liquidation_nonce={}
```
`src/v2/liquidation.rs:793` — one per deal in a taker bulk packet (tag 103).
Literals `order_seq=0` and `trading_fee_base=0` (no close fee on liquidation).
`maker` is the holder pubkey or the literal `none` for public-pool deals.

```
event=SurplusToRisk protocol_version={} slot={} timestamp={} event_index={} history_seq={} trader={} settlement={} pair_id={} amount_base={}
```
`src/v2/liquidation.rs:861` — residual account equity swept to the risk fund at
finalize. Note: `history_seq` carries the **liquidation nonce** here.

```
event=BulkLiquidation protocol_version={} slot={} timestamp={} event_index={} trader={} settlement={} receipt={} nonce={} deal_count={} pair_mask={}
event=LiquidationBadDebtItem protocol_version={} slot={} timestamp={} event_index={} trader={} settlement={} receipt={} nonce={} history_seq={} deal_id={} bad_debt_base={}
event=LiquidationReceiptAudit protocol_version={} slot={} timestamp={} event_index={} trader={} settlement={} receipt={} nonce={} deal_count={} outcomes_len={} pair_mask={} all_deals_done={} outcomes_hash={} packet_bad_debt_base={} rent_lamports={}
```
`src/v2/liquidation.rs:903`, `:918` (one per packet deal), `:934` — tag 103 packet
summary. `outcomes_hash` = 64-char hex SHA-256 audit hash
(domain `b"domi_v2:liquidation_receipt_audit:v1"`, liquidation.rs:144-176).

```
event=DealRentReclaimed protocol_version={} slot={} timestamp={} event_index={} settlement={} deal_id={} reason=2 recipient={}
event=DealRentReclaimedBatch protocol_version={} slot={} timestamp={} event_index={} settlement={} reason={} recipient={} reclaimed_count={} reclaimed_lamports={}
event=LiquidationReceiptRentReclaimed protocol_version={} slot={} timestamp={} event_index={} trader={} settlement={} receipt={} nonce={} recipient={} lamports={} mode=1
```
`src/v2/liquidation.rs:955`, `:967` / `:1605`, `:989`. `reason`: 2 = taker
liquidation, 3 = maker liquidation. Rent recipient is the keeper.
`LiquidationReceiptRentReclaimed` only fires when the settlement's
receipt-ephemeral-close feature flag is on.

```
event=MakerDealLiquidated protocol_version={} slot={} timestamp={} event_index={} history_seq={} deal_id={} maker={} taker={} settlement={} page_id={} maker_slot={} pair_id={} direction={} amount_e9={} oracle_price_e9={} execution_price_e9={} leverage_e9={} margin_base={} raw_loss_base={} maker_debit_base={} risk_paid_base={} risk_surplus_base={} bad_debt_base={} credited_gain_base={} released_taker_margin_base={} position_size_e9={} position_value_e9={} position_margin_base={}
event=MakerLiquidationBatch protocol_version={} slot={} timestamp={} event_index={} settlement={} page_id={} pair_id={} candidate_count={} auto_added_count={} force_closed_count={}
```
`src/v2/liquidation.rs:1536` (per force-closed deal) and `:1590` (packet summary) —
tag 105. `execution_price_e9` may differ from `oracle_price_e9` when the credited
gain was capped (`effective_force_close_price`).

```
event=RiskFundDeposited protocol_version={} slot={} timestamp={} event_index=0 admin={} settlement={} pair_id={} amount_base={} risk_fund_after_base={}
event=RiskFundWithdrawn protocol_version={} slot={} timestamp={} event_index=0 admin={} settlement={} pair_id={} amount_base={} risk_fund_after_base={}
```
`src/v2/liquidation.rs:1697` — tags 106/107.

---

## Indexer Guidance

- Match on the `DOMI_V2_EVENT ` prefix, then split on spaces; the first token is
  `event=<Name>`. Some values are literals (`scope=`, `source=`, `reason=`,
  `order_seq=0`, `trading_fee_base=0`, `page_id=0`, `mode=1`) — treat them as part
  of the event schema.
- `OrderTerminated` and `OrderConverted` have multiple field layouts; dispatch on
  the presence of `reason=` / `margin_base=` and the `slot=` field.
- `event_index` is only unique within one instruction execution; combine with the
  transaction signature for a primary key.
