---
id: v2-errors
title: V2 Error Codes (400–461)
sidebar_position: 5
---

# V2 Error Codes (400–461)

V2 errors are **append-only** in `PerpetualOptionsError`; legacy numeric codes are
unchanged (`src/error.rs:404`). Every entry below is transcribed verbatim from
`src/error.rs:405-528` (enum variants + `#[error]` messages). The on-chain log lines
emitted by `print()` match these messages (`src/error.rs:759-821`).

On-chain, a custom error surfaces as `Custom(N)` in the transaction logs, where `N` is
the code below.

---

| Code | Variant | Message | Typical cause |
| --- | --- | --- | --- |
| 400 | `V2InvalidVersion` | Invalid V2 account version | account `version != CURRENT_VERSION (1)` |
| 401 | `V2InvalidSettlement` | Invalid V2 settlement | settlement PDA/mint mismatch; zero mint/custody; settlement not ACTIVE at bootstrap |
| 402 | `V2InvalidPair` | Invalid V2 fast-lane pair | `pair_id` outside 1–3 |
| 403 | `V2InvalidOraclePolicy` | Invalid V2 oracle policy | market/position oracle policy ≠ `ORACLE_POLICY_DOMI_CUSTOS_V1 (1)` |
| 404 | `V2PriceSetIncomplete` | V2 price set is incomplete | missing/extra/misordered oracle price accounts (one per unique active pair, first-occurrence order) |
| 405 | `V2StaleNonce` | Stale V2 nonce | liquidation packet nonce ≠ `trader.liquidation_nonce` / `receipt_nonce ≠ nonce+1` |
| 406 | `V2ReceiptReplay` | V2 receipt replay | receipt PDA belongs to another trader or is already closed |
| 407 | `V2MissingFeeAccounts` | Missing V2 fee accounts | fee bundle missing or a required referral commission/top-agent account is absent |
| 408 | `V2CapacityExceeded` | V2 account capacity exceeded | maker/deal/order/market vector full; `order_seq` exceeds trigger-deal mask; sequence overflow |
| 409 | `V2ConservationFailure` | V2 conservation invariant failed | ledger conservation/bitmap/invariant check failed |
| 410 | `V2MarginTooSmall` | V2 margin rounds to zero native units | frozen margin rescales to 0 base units at fill price |
| 411 | `V2InvalidLifecycle` | Invalid V2 lifecycle state | lifecycle state machine violation |
| 412 | `V2AccountMismatch` | V2 account relation mismatch | account's mint/owner/pair/page fields don't match the expected relation |
| 413 | `V2InvalidAccountKind` | Invalid V2 account kind | header `is_initialized == false` or `account_kind` mismatch on unpack |
| 414 | `V2Paused` | V2 protocol or settlement is paused | protocol `paused` or settlement `status != SETTLEMENT_ACTIVE` on a gated instruction |
| 415 | `V2TraderLiquidationLocked` | V2 trader is locked by liquidation | trader status is LIQUIDATING/LIQUIDATED on an action that requires ACTIVE |
| 416 | `V2WithdrawAtRisk` | V2 withdrawal would leave insufficient equity | withdraw amount + net unrealized loss exceeds available |
| 417 | `V2InvalidFeeRecipient` | Invalid V2 fee recipient | recipient default/equal/off-curve, or FeeRecipients not locked |
| 418 | `V2NothingToClaim` | No V2 fees are claimable | accrued/claimable balance is zero |
| 419 | `V2InvalidReferral` | Invalid V2 referral chain | self-bind, default/off-curve key, or invitee/inviter mismatch |
| 420 | `V2PermissionlessDisabled` | Permissionless V2 pool creation is disabled | `protocol.reserved[1] == 0` on a permissionless instruction |
| 421 | `V2PublicPoolNotConfigured` | V2 public pool is not configured | `settlement.public_page_id == u16::MAX` |
| 422 | `V2PoolInsolvent` | V2 public pool is insolvent | public pool effective equity is 0 while shares are outstanding |
| 423 | `V2MakerFrozen` | V2 maker slot is frozen | withdraw from a maker slot with `MAKER_FLAG_FROZEN` set |
| 424 | `V2MakerNotLiquidatable` | V2 maker slot is not liquidatable | `LiquidateLpV2` on a maker with `locked_base <= equity_base` |
| 425 | `V2CannotModifyPublicSlot` | Cannot modify the V2 public maker slot through the private setter | `SetMakerParamsV2` targeting the public slot |
| 426 | `V2PrivateOnPublicSlot` | V2 private operation targets the public maker slot | private liquidity/margin operation aimed at the public slot |
| 427 | `V2PublicSlotNotContaminated` | V2 public slot is not contaminated | repair requested but the public slot has no stray private liquidity |
| 428 | `V2PublicSlotRepairUnsafe` | V2 public slot repair is unsafe | contaminated liquidity is not inert (locked/maintenance ≠ 0, equity ≠ available, or frozen) |
| 429 | `V2OrderNotPending` | V2 order is not pending | order not open / wrong taker / trigger binding mismatch |
| 430 | `V2OrderExpired` | V2 order has expired | `now > good_till` (trigger), or `now <= good_till` when expiring, or deadline passed at make time |
| 431 | `V2OrderSeqMismatch` | V2 order sequence mismatch | `args.order_seq != sequence.next_order_seq` |
| 432 | `V2OrderKindDisabled` | V2 order kind is disabled | order-kind bit disabled by market override or protocol switch |
| 433 | `V2OrderAmountBelowMin` | V2 order amount is below the market minimum | open amount < `min_order_amount_e9` (make-open only) |
| 434 | `V2RewardGasInsufficient` | V2 native keeper reward is insufficient | `reward_gas == 0` or < `minimum_reward_gas`; or order PDA can't cover the payout |
| 435 | `V2TriggerNotMet` | V2 order trigger condition is not met | limit/stop/take price condition not satisfied |
| 436 | `V2PriceDeviationTooLarge` | V2 market-open price deviation is too large | market-open fill deviates from target beyond `max_market_open_deviation_e9` |
| 437 | `V2OrderKindMismatch` | V2 order kind or offset mismatch | trigger instruction used against the wrong order kind/offset |
| 438 | `V2InvalidOrderDerivedDeal` | V2 order-derived Deal id is invalid | trigger-derived deal id (`0xC000… \| order_seq`) invalid |
| 439 | `V2PositionNotFound` | V2 position was not found | no active position slot for (pair, direction) |
| 440 | `V2CloseExceedsAvailable` | V2 close amount exceeds available position size | close amount > position `size_e9 − freeze_e9` |
| 441 | `V2PendingOrdersFull` | V2 pending-order capacity exceeded | `pending_order_ids` already holds 32 entries |
| 442 | `V2PositionSlotsFull` | V2 position-slot capacity exceeded | 8 position slots occupied |
| 443 | `V2DealsPerPositionFull` | V2 deals-per-position capacity exceeded | position already tracks 50 deals |
| 444 | `V2InvalidMakerSlot` | V2 maker slot is invalid | maker slot empty / `reject_order` / frozen at fill time |
| 445 | `V2InsufficientMakerAvailable` | V2 maker liquidity is insufficient | maker cannot cover margin (+ maintenance) or the taker gain |
| 446 | `V2DealPageMismatch` | V2 Deal/page/order binding mismatch | deal/page/position/slot binding inconsistent |
| 447 | `V2CloseExceedsRemaining` | V2 close amount exceeds Deal remaining size | computed close amount is 0 against the deal's remaining size |
| 448 | `V2OrderNotOrphan` | V2 order is not orphaned | cleanup target fails the orphan criteria (close order on an empty position) |
| 449 | `V2TraderLiquidatingCancel` | Liquidating V2 trader orders require liquidation cancellation | normal cancel path used while the trader is being liquidated |
| 450 | `V2MarketTemplateEmpty` | V2 market template is empty | bootstrap with an empty template or a fast-lane pair missing from it |
| 451 | `V2UnsupportedConditionalChain` | V2 conditional order chain is unsupported | stop-take open with a limit price chained to TP/SL |
| 452 | `V2InvalidFeeAccountBundle` | V2 fee account bundle is invalid | fee bundle accounts structurally invalid |
| 453 | `V2DealNotDetached` | V2 Deal is not terminal and detached | deal reclaim requested while the deal is still referenced or its slot is not free |
| 454 | `V2NotLiquidatable` | V2 account is not liquidatable | first liquidation packet but account is not at risk |
| 455 | `V2BulkDealCapExceeded` | V2 bulk deal cap exceeded | > 32 deals in a taker packet / > 4 deals in a maker packet / empty set |
| 456 | `V2DuplicateDeal` | V2 duplicate deal in liquidation packet | duplicate or out-of-order deal ids in a packet |
| 457 | `V2IncompleteDealTail` | V2 liquidation packet does not cover the declared deal set | dynamic account tail doesn't match `expected_deal_count` / `page_groups` |
| 458 | `V2InvalidPageGroups` | V2 page groups are invalid | page ids not strictly increasing, or counts don't sum to `expected_deal_count` |
| 459 | `V2ReceiptFull` | V2 liquidation receipt is full | more than 32 outcomes in one receipt |
| 460 | `V2TraderNotLiquidated` | V2 trader is not in the liquidated state | `LiquidateCancelOrdersV2` before the trader reached TRADER_LIQUIDATED |
| 461 | `V2InsufficientRiskFund` | V2 risk fund balance is insufficient | risk-fund withdrawal above `risk_fund_equity_base` |

---

## Notes

- Codes 400–461 never overlap the legacy range; a client can branch on `code >= 400`
  to select the V2 error table.
- Many V2 instructions also return **legacy** errors for generic failures
  (`InvalidPDA`, `Unauthorized`, `NotAuthorizedKeeper`, `InvalidAmount`,
  `InsufficientFunds`, `SlippageExceeded`, `MathOverflow`, …). See the legacy
  [Error Reference](../errors/overview.md) for those.
- `ProgramError::MissingRequiredSignature`, `InvalidAccountData`,
  `IncorrectProgramId` and `InvalidArgument` surface as native (non-custom) errors.
