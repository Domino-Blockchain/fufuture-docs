---
id: v2-instructions
title: V2 Instruction Reference (Tags 60–107)
sidebar_position: 2
---

# V2 Instruction Reference (Tags 60–107)

V2 instructions are Borsh-serialized variants of the same `PerpetualInstruction`
enum as the legacy lane: `<u8 tag> + <borsh args>`. Tag assignments are in
`src/instructions/mod.rs:1594-1641`; dispatch is in `src/processor/mod.rs:894-1109`;
argument structs are in `src/v2/instruction.rs`.

**Same Program ID as legacy.** Legacy tags 0–59 are unchanged. V2 account layouts are
not compatible with legacy accounts — see [V2 Overview](./overview.md).

Conventions for the account tables below:

- **[signer]** — the handler requires a signature (`MissingRequiredSignature` otherwise).
- **[writable]** — the account data or lamports are mutated (pack, creation, or SPL transfer).
- PDA seeds are listed without the leading `b"domi_v2"` namespace; see
  [V2 PDAs](./pda.md) for the full seed table.
- "Fee bundle" refers to the fixed 7-account referral/fee tail described under tag 96.
- Event names link to [V2 Events](./events.md); error codes to
  [V2 Errors](./errors.md).

---

## Protocol & Market Configuration

### 60 — InitializeProtocolV2

Args `InitializeProtocolV2Args` (instruction.rs:8): `admin: Pubkey`,
`maintenance_margin_rate_e9: u64`, `min_deposit_amount: u64` (base units).

| # | Account | Flags | Notes |
| --- | --- | --- | --- |
| 0 | payer | [signer][writable] | pays rent |
| 1 | protocol PDA `["protocol"]` | [writable] | created |
| 2 | system program | | |

- Permission: none (first-call setup). Pause gate: none.
- Checks: `maintenance_margin_rate_e9 ∈ (0, 1e9]` → `InvalidParameter`;
  PDA key → `InvalidPDA`; already initialized → `AccountAlreadyInitialized`.
- Event: `ProtocolInitialized` (processor.rs:129).

### 61 — SetProtocolMaintenanceMarginRateV2

Args `SetMaintenanceMarginRateV2Args` (instruction.rs:15): `maintenance_margin_rate_e9: u64`.

| # | Account | Flags |
| --- | --- | --- |
| 0 | admin | [signer] |
| 1 | protocol PDA | [writable] |

- Permission: **admin** (`require_admin` → `Unauthorized`). Pause gate: none.
- Checks: value ∈ (0, 1e9] → `InvalidParameter`.
- Event: `MaintenanceMarginRateUpdated scope=protocol`.

### 62 — SetSettlementMaintenanceMarginRateV2

Args: same as tag 61. Settlement scope accepts **0 = inherit protocol default**.

| # | Account | Flags |
| --- | --- | --- |
| 0 | admin | [signer] |
| 1 | protocol PDA | |
| 2 | settlement mint | |
| 3 | settlement PDA `["settlement", mint]` | [writable] |

- Permission: **admin**. Pause gate: none.
- Checks: value ≤ 1e9 → `InvalidParameter`; settlement mint mismatch → `V2InvalidSettlement` (401).
- Event: `MaintenanceMarginRateUpdated scope=settlement`.

### 63 — SetUserLeverageV2

Args `SetUserLeverageV2Args` (instruction.rs:22): `pair_id: u16`, `leverage_e9: u64`.

| # | Account | Flags |
| --- | --- | --- |
| 0 | owner | [signer][writable] | pays rent on first use |
| 1 | settlement mint | |
| 2 | market catalog PDA `["markets", mint, 0u16le]` | |
| 3 | user-leverage PDA `["ulev", mint, pair, owner]` | [writable] | created if absent |
| 4 | system program | |

- Permission: **owner**. Pause gate: none.
- Checks: `pair_id ∈ 1..=3` → `V2InvalidPair` (402); leverage ∈ [1e9, 1000e9] and within
  market `[min_leverage_e9, max_leverage_e9]` → `InvalidLeverage`; pair must exist →
  `TradingPairNotFound`.
- Event: `LeverageSet`.

### 75 — SetPermissionlessMarketsV2

Args `SetPermissionlessMarketsV2Args` (instruction.rs:46): `enabled: bool`.

Accounts: `admin` [signer], `protocol` PDA [writable]. Permission: **admin**.
Writes `protocol.reserved[1]`. Event: `PermissionlessMarketsUpdated` (pool.rs:394).

### 88 — SetKeeperV2

Args `SetKeeperV2Args` (instruction.rs:93): `keeper: Pubkey`, `enabled: bool`.

Accounts: `admin` [signer], `protocol` PDA [writable]. Permission: **admin**.
Checks: duplicate → `KeeperAlreadyExists`; > 16 keepers → `TooManyKeepers`;
disable of unknown keeper → `KeeperNotFound`. Event: `KeeperUpdated` (pool.rs:417).

### 89 — UpsertMarketTemplateV2

Args `UpsertMarketTemplateV2Args` (instruction.rs:99): `entry: MarketEntryV2`
(see [PDA doc](./pda.md#marketentryv2--embedded-max-97-bytes-staters333-375)).

| # | Account | Flags |
| --- | --- | --- |
| 0 | admin | [signer][writable] | payer on creation |
| 1 | protocol PDA | |
| 2 | market-template PDA `["market-template"]` | [writable] | created if absent |
| 3 | system program | |

- Permission: **admin**. Checks: `entry.validate()` (oracle policy = 1 →
  `V2InvalidOraclePolicy` 403, feed id non-zero → `InvalidPriceFeed`, leverage/fee
  ranges → `InvalidParameter`); extra accounts → `InvalidArgument`.
- Event: `MarketTemplateUpserted` (market.rs:93).

### 90 — BootstrapMarketsV2

No args. Creates the whole per-settlement account set for fast-lane pairs 1–3 from
the template.

| # | Account | Flags |
| --- | --- | --- |
| 0 | payer | [signer][writable] |
| 1 | protocol PDA | |
| 2 | market-template PDA | |
| 3 | settlement mint | |
| 4 | settlement PDA | [writable] | must already exist (tag 76/77) |
| 5 | market catalog PDA `["markets", mint, 0u16le]` | [writable] | create-or-load |
| 6 | sequence PDA `["seq", mint]` | [writable] | fresh: deal_seq=1, order_seq=0, history_seq=0 |
| 7–9 | market-risk PDAs `["risk", mint, pair]` for pairs 1,2,3 | [writable] | create-or-load |
| 10–12 | market-order-config PDAs `["order-market", mint, pair]` for pairs 1,2,3 | [writable] | fresh defaults: leverage 100e9, min amount 1, deviation 5%, min reward 3_000_000 |
| 13 | system program | |

- Permission: **permissionless**, but requires `protocol.permissionless_markets()` →
  `V2PermissionlessDisabled` (420).
- Checks: template empty or a pair 1–3 missing → `V2MarketTemplateEmpty` (450);
  existing accounts cross-checked → `V2AccountMismatch` (412).
- Event: `MarketsBootstrapped` (market.rs:400).

### 91 — SetMarketOrderConfigV2

Args `SetMarketOrderConfigV2Args` (instruction.rs:104): `pair_id: u16`,
`default_leverage_e9: u64`, `min_order_amount_e9: u128`, `order_override_mask: u8`,
`order_override_value: u8`, `max_market_open_deviation_e9: u64`,
`minimum_reward_gas: u64` (lamports).

Accounts: `admin` [signer], `protocol` PDA, `mint`, catalog PDA,
market-order-config PDA [writable].

- Permission: **admin**.
- Checks: pair 1–3 → `V2InvalidPair`; market exists → `TradingPairNotFound`;
  default leverage within market min/max, `min_order_amount_e9 != 0`, mask/value ⊆
  `0b111`, deviation ≤ 1e9 → `InvalidParameter`. `minimum_reward_gas` is unbounded.
- Event: `MarketOrderConfigUpdated` (market.rs:465).

### 92 — SetOrderSwitchV2

Args `SetOrderSwitchV2Args` (instruction.rs:115): `order_switch: u8`
(bits: market `0b001`, limit `0b010`, stop-take `0b100`; stored 0 = all enabled).

Accounts: `admin` [signer], `protocol` PDA [writable]. Permission: **admin**.
Checks: `order_switch & !0b111 != 0` → `InvalidParameter`.
Event: `OrderSwitchUpdated` (market.rs:504).

---

## Trader Funds

### 64 — InitializeTraderMarginV2

No args.

| # | Account | Flags |
| --- | --- | --- |
| 0 | owner | [signer][writable] | payer |
| 1 | settlement mint | |
| 2 | settlement PDA | |
| 3 | trader PDA `["trader", mint, owner]` | [writable] | created |
| 4 | system program | |

- Permission: **owner**. Pause gate: none.
- Event: `TraderMarginInitialized` (funds.rs:322).

### 65 — DepositMarginV2

Args `AmountBaseV2Args` (instruction.rs:28): `amount_base: u64` (settlement base units).

| # | Account | Flags |
| --- | --- | --- |
| 0 | owner | [signer] | SPL transfer authority |
| 1 | settlement mint | |
| 2 | protocol PDA | |
| 3 | settlement PDA | |
| 4 | trader PDA | [writable] |
| 5 | owner token account | [writable] | debit |
| 6 | custody PDA `["custody", mint]` | [writable] | credit |
| 7 | SPL token program | |

- Permission: **owner**. Pause gate: protocol `paused` **or** settlement not ACTIVE → `V2Paused` (414).
- Checks: `amount_base > 0` → `InvalidAmount`; mint decimals match → `InvalidTokenMint`;
  amount ≥ min(settlement.min_deposit_amount ‖ protocol) → `DepositAmountTooSmall`;
  trader LIQUIDATING → `V2TraderLiquidationLocked` (415); trader LIQUIDATED with
  pending orders → 415 (deposit resets LIQUIDATED → ACTIVE when no pending orders).
- SPL: `transfer_checked(owner token → custody, authority=owner)`.
- Event: `MarginDeposited` (funds.rs:409).

### 66 — WithdrawMarginV2

Args: `AmountBaseV2Args`.

| # | Account | Flags |
| --- | --- | --- |
| 0 | owner | [signer] |
| 1 | settlement mint | |
| 2 | protocol PDA | |
| 3 | settlement PDA | |
| 4 | market catalog PDA (page 0) | |
| 5 | trader PDA | [writable] |
| 6 | owner token account | [writable] | credit |
| 7 | custody-authority PDA `["custody-auth", mint]` | |
| 8 | custody PDA | [writable] | debit |
| 9 | SPL token program | |
| 10… | oracle price accounts | | one per unique active pair, in first-occurrence order of `trader.positions` |

- Permission: **owner**. Pause gate: protocol `paused` → `V2Paused`
  (**settlement pause deliberately does NOT block withdrawals**, funds.rs:448-450).
- Checks: trader must be ACTIVE → `V2TraderLiquidationLocked`;
  `available_base ≥ amount` → `InsufficientBalance`; price-set completeness →
  `V2PriceSetIncomplete` (404); risk gate `available < amount + net_unrealized_loss`
  → `V2WithdrawAtRisk` (416).
- SPL: `transfer_checked(custody → owner token)` signed by custody-authority PDA.
- Event: `MarginWithdrawn` (funds.rs:516).

---

## Fees & Referral

### 67 — InitializeFeeRecipientsV2

Args `SetFeeRecipientsV2Args` (instruction.rs:34): `platform_owner: Pubkey`,
`trade_fee_owner: Pubkey`.

Accounts: `admin` [signer][writable] (payer), `protocol` PDA,
fee-recipients PDA `["fee-recipients"]` [writable, created], system program.

- Permission: **admin**. Checks: recipients valid (non-default, distinct, on-curve)
  → `V2InvalidFeeRecipient` (417). Event: `FeeRecipientsInitialized` (fee.rs:263).

### 68 — SetFeeRecipientsV2

Args: `SetFeeRecipientsV2Args`. Accounts: `admin` [signer], `protocol` PDA,
fee-recipients PDA [writable]. Permission: **admin**. Same recipient validation.
Event: `FeeRecipientsUpdated` (fee.rs:293).

### 69 — InitializeCommissionAccountV2

No args.

Accounts: `payer` [signer][writable], `owner` (beneficiary, **not** signer),
settlement mint, commission PDA `["commission", mint, owner]` [writable, created],
system program.

- Permission: permissionless (payer signs). Checks: owner non-default on-curve →
  `Unauthorized`. **No event emitted.**

### 70 — BindInviterV2

No args. Mint-agnostic (no settlement account).

Accounts: `invitee` [signer][writable] (payer), `inviter` (not signer),
invite-relation PDA `["invite", invitee]` [writable, created], system program.

- Checks: self-bind / default / off-curve → `V2InvalidReferral` (419).
- Event: `InviterBound` (fee.rs:395).

### 71 — SetTopAgentV2

Args `SetTopAgentV2Args` (instruction.rs:40): `agent: Pubkey`, `approved: bool`.

Accounts: `admin` [signer][writable], `protocol` PDA, settlement mint,
top-agent PDA `["top-agent", mint, agent]` [writable, create-or-load], system program.

- Permission: **admin**. Checks: agent default/off-curve → `V2InvalidReferral`.
- Event: `TopAgentUpdated` (fee.rs:465).

### 72 — ClaimPlatformFeesV2 / 73 — ClaimTradeFeesV2

No args. Both delegate to the same handler with `fee_kind` 1 (platform) / 2 (trade).

| # | Account | Flags |
| --- | --- | --- |
| 0 | owner | [signer] | must equal `platform_owner` (72) / `trade_fee_owner` (73) |
| 1 | protocol PDA | |
| 2 | fee-recipients PDA | |
| 3 | settlement mint | |
| 4 | settlement PDA | [writable] | accrued counter zeroed |
| 5 | recipient token account (of owner) | [writable] |
| 6 | custody-authority PDA | |
| 7 | custody PDA | [writable] |
| 8 | SPL token program | |

- Permission: **recipient owner** (not admin). Pause gate: protocol `paused` → `V2Paused`.
- Checks: accrued = 0 → `V2NothingToClaim` (418). Counter zeroed before transfer.
- Event: `ProtocolFeeClaimed` (fee.rs:545).

### 74 — ClaimCommissionV2

No args.

Accounts: `owner` [signer], settlement mint, `protocol` PDA, settlement PDA,
commission PDA `["commission", mint, owner]` [writable], recipient token account
[writable], custody-authority PDA, custody PDA [writable], SPL token program.

- Permission: **commission owner**. Pause gate: protocol `paused` → `V2Paused`.
- Checks: `accrued − claimed == 0` → `V2NothingToClaim`.
- Event: `CommissionClaimed` (fee.rs:624).

---

## Liquidity

### 76 — CreatePublicPoolV2

No args. Creates (if needed) the settlement namespace and configures the public pool
on canonical page 0; the public maker slot holder is the custody-authority PDA.

| # | Account | Flags |
| --- | --- | --- |
| 0 | payer | [signer][writable] |
| 1 | protocol PDA | |
| 2 | settlement mint | |
| 3 | custody-authority PDA | |
| 4 | settlement PDA | [writable] | created if absent |
| 5 | custody PDA | [writable] | created (system create + `initialize_account3`) if absent |
| 6 | liquidity page 0 PDA `["liquidity", mint, 0u16le]` | [writable] | created if absent |
| 7 | SPL token program | |
| 8 | system program | |

- Permission: **permissionless** — requires `permissionless_markets()` →
  `V2PermissionlessDisabled` (420). Pause gate: none (fresh settlement must be ACTIVE
  → `V2InvalidSettlement`).
- Checks: first configuration requires all public counters zero →
  `V2ConservationFailure` (409); repeat calls must match page 0 / custody-authority
  holder → `V2AccountMismatch`.
- Events: `SettlementInitialized` (if new), `PublicPoolConfigured` (pool.rs:487/498).

### 77 — CreatePrivatePoolV2

Args `CreatePrivatePoolV2Args` (instruction.rs:51): `page_id: u16`, `amount_base: u64`.

Accounts: `holder` [signer][writable] (payer + SPL authority), `protocol` PDA, mint,
custody-authority PDA, settlement PDA [writable, maybe created], liquidity page
`page_id` PDA [writable, maybe created], holder token account [writable, debit],
custody PDA [writable, maybe created, credit], SPL token program, system program.

- Permission: permissionless (holder signs); requires `permissionless_markets()`.
- Checks: `amount_base > 0` → `InvalidAmount`; ≥ min deposit → `DepositAmountTooSmall`;
  no maker slot free → `V2CapacityExceeded` (408); may never occupy the public slot →
  `V2PrivateOnPublicSlot` (426).
- SPL: `transfer_checked(holder → custody)`.
- Events: `SettlementInitialized` (if new), `PrivateLiquidityProvided` (pool.rs:597).

### 78 — InitializeLiquidityPageV2

Args `InitializeLiquidityPageV2Args` (instruction.rs:57): `page_id: u16`.

Accounts: `payer` [signer][writable], `protocol` PDA, mint, settlement PDA [writable,
must exist], liquidity page PDA [writable, created], system program.

- Permission: permissionless (any payer). Raises `settlement.liquidity_page_count`.
- Event: `LiquidityPageInitialized` (pool.rs:657).

### 79 — InitializePublicShareV2

No args. Accounts: `holder` [signer][writable] (payer), mint, settlement PDA,
public-share PDA `["pub-share", mint, holder]` [writable, created], system program.

- Checks: public pool configured → `V2PublicPoolNotConfigured` (421).
- Event: `PublicShareInitialized` (pool.rs:707).

### 80 — ProvideLiquidityV2

Args `LiquidityAmountV2Args` (instruction.rs:62): `page_id: u16`, `maker_slot: u8`,
`amount: u64` (base units), `pool_kind: u8` (1 public / 2 private),
`limit_out: u64` (min shares out on public path).

Accounts: `holder` [signer], mint, `protocol` PDA, settlement PDA [writable on public
path], liquidity page PDA [writable on private path], market catalog page 0,
holder token account [writable, debit], custody PDA [writable, credit],
SPL token program; **public path only**: public-share PDA [writable], then oracle
price accounts (one per active pair).

- Permission: holder. Pause gate: protocol `paused` or settlement not ACTIVE → `V2Paused`.
- Checks: `amount > 0`, valid pool kind → `InvalidAmount`; min deposit →
  `DepositAmountTooSmall`. Private: not the public slot → `V2PrivateOnPublicSlot`;
  maker occupied & owned by holder → `MakerNotFound`/`UnauthorizedMaker`; **no extra
  accounts allowed** → `V2PriceSetIncomplete`. Public: configured/page/slot match →
  `V2PublicPoolNotConfigured`/`V2AccountMismatch`; equity 0 with shares outstanding →
  `V2PoolInsolvent` (422); `shares < limit_out` → `SlippageExceeded`.
- Events: `PrivateLiquidityProvided` (private, pool.rs:791) / `PublicShareMinted`
  (public, pool.rs:869).

### 81 — WithdrawLiquidityV2

Args: `LiquidityAmountV2Args` (`amount` = base units private / shares public;
`limit_out` = min payout).

Accounts: `holder` [signer], mint, `protocol` PDA, settlement PDA [writable],
liquidity page PDA [writable], catalog page 0, holder token account [writable,
credit], custody-authority PDA, custody PDA [writable, debit], SPL token program;
**public path**: public-share PDA [writable]; then oracle price accounts
(public path and private path both consume prices).

- Permission: holder. Pause gate: protocol `paused` → `V2Paused` (settlement pause
  does not block withdrawals).
- Checks (private): frozen maker → `V2MakerFrozen` (423, checked **before** prices);
  ownership → `MakerNotFound`/`UnauthorizedMaker`; `available < amount + net_loss` or
  `equity < amount` → `V2WithdrawAtRisk`; `amount < limit_out` → `SlippageExceeded`;
  non-zero dust below min deposit → `RemainingAmountTooSmall`; full drain frees the
  slot. Checks (public): `share.shares < amount` → `InsufficientShares`;
  `payout < limit_out` → `SlippageExceeded`; `payout > public_available/equity` →
  `InsufficientLiquidity`.
- SPL: custody → holder, signed by custody authority.
- Event: `LiquidityWithdrawn` (pool.rs:1055).

### 82 — SetMakerParamsV2

Args `MakerParamsV2Args` (instruction.rs:71): `page_id`, `maker_slot`, plus optional
fields `leverage_e9`, `mmr_e9`, `add_margin_rate_e9`, `auto_add_margin`,
`reject_order` (`None` = unchanged).

Accounts: `holder` [signer], mint, settlement PDA, liquidity page PDA [writable].

- Permission: maker holder. Checks: target = public slot → `V2CannotModifyPublicSlot`
  (425); leverage ∈ [1e9, 1000e9] → `InvalidLeverage`; mmr/add-margin ≤ 1e9 →
  `InvalidParameter`.
- Event: `MakerParamsUpdated` (pool.rs:1132).

### 83 — SetEscrowParamsV2

Args: `MakerParamsV2Args`. The admin-only path to tune the **public** (escrow) maker
slot; `page_id`/`maker_slot` args are ignored in favor of `settlement.public_*`.

Accounts: `admin` [signer], `protocol` PDA, mint, settlement PDA, public liquidity
page PDA [writable].

- Permission: **admin**. Checks: public pool configured → `V2PublicPoolNotConfigured`;
  same range checks as tag 82.
- Event: `MakerParamsUpdated` (authority = admin, pool.rs:1190).

### 84 — AddMakerMarginV2

Args `MakerMarginV2Args` (instruction.rs:82): `deal_id: u64`, `amount_base: u64`.
Moves margin from the maker's **TraderMargin** account into a live deal (pure ledger,
no SPL).

Accounts: `maker` [signer], mint, settlement PDA, liquidity page PDA [writable]
(PDA re-derived from `page.page_id`), maker's trader PDA `["trader", mint, maker]`
[writable].

- Checks: `amount > 0` → `InvalidAmount`; deal active → `DealNotActive`;
  `trader.available/equity < amount` → `InsufficientBalance`.
- Event: `MakerMarginAdded source=trader` (pool.rs:1278).

### 85 — AddMakerMarginFromPoolV2

Args: `MakerMarginV2Args`. Moves margin from the maker slot's own `available_base`.

Accounts: `maker` [signer], mint, settlement PDA, liquidity page PDA [writable].

- Checks: deal on public slot → `V2PrivateOnPublicSlot`;
  `maker.available < amount` → `InsufficientLiquidity`.
- Event: `MakerMarginAdded source=pool` (pool.rs:1347).

### 86 — LiquidateLpV2

Args `LiquidateLpV2Args` (instruction.rs:88): `maker_slot: u8`. **Freeze only** — sets
`reject_order = true` and `MAKER_FLAG_FROZEN`; funds move later via tag 105.

Accounts: `keeper` [signer], `protocol` PDA, mint, settlement PDA, liquidity page PDA
[writable].

- Permission: **keeper** (`protocol.is_keeper` → `NotAuthorizedKeeper`). Pause gate:
  none (liquidation paths are intentionally not gated).
- Checks: `locked_base <= equity_base` → `V2MakerNotLiquidatable` (424).
- Event: `LpFrozen` (pool.rs:1404).

### 87 — RepairPublicSlotPrivateLiquidityV2

No args. Admin repair for private liquidity booked onto the public slot.

Accounts: `admin` [signer], `protocol` PDA, mint, custody-authority PDA, settlement
PDA, public liquidity page PDA [writable].

- Permission: **admin**.
- Checks: not contaminated → `V2PublicSlotNotContaminated` (427); stray liquidity not
  inert (locked/maintenance ≠ 0, equity ≠ available, or frozen) →
  `V2PublicSlotRepairUnsafe` (428); no free private slot → `V2CapacityExceeded`.
- Event: `PublicSlotPrivateLiquidityRepaired` (pool.rs:1455).

---

## Orders (two-phase: make → keeper trigger)

### 93 — MakeOpenOrderV2

Args `MakeOpenOrderV2Args` (instruction.rs:120): `pair_id: u16`, `direction: u8`
(1 long / 2 short), `target_price_e9: u64`, `amount_e9: u128`, `reward_gas: u64`
(lamports), `order_seq: u64`, `good_till: i64`, `deadline: i64`, `is_market: bool`.

| # | Account | Flags |
| --- | --- | --- |
| 0 | taker | [signer][writable] | pays rent + reward escrow |
| 1 | settlement mint | |
| 2 | protocol PDA | |
| 3 | settlement PDA | |
| 4 | market catalog PDA (page 0) | |
| 5 | market-order-config PDA | |
| 6 | trader PDA | [writable] |
| 7 | user-leverage PDA | [writable] | created (default leverage) if absent |
| 8 | sequence PDA | [writable] |
| 9 | order PDA `["order", mint, order_seq]` | [writable] | created |
| 10 | trigger PDA `["trigger", mint, order_seq]` | [writable] | created |
| 11 | system program | |

- Permission: taker. Pause gate: protocol/settlement → `V2Paused`; market not ACTIVE
  → `TradingPaused`; order-kind switch bit → `V2OrderKindDisabled` (432).
- Checks: `target_price_e9 > 0` → `InvalidPrice`; `deadline != 0 && now > deadline` →
  `V2OrderExpired` (430); `amount ≥ min_order_amount_e9` → `V2OrderAmountBelowMin`
  (433, **make-open only**); `reward_gas ≥ minimum_reward_gas` →
  `V2RewardGasInsufficient` (434); `order_seq == sequence.next_order_seq` →
  `V2OrderSeqMismatch` (431); trader ACTIVE → `V2TraderLiquidationLocked`;
  `available < margin+fee` → `InsufficientFunds`; 32 pending-order cap →
  `V2PendingOrdersFull` (441). `good_till <= 0` defaults to now + 7 days.
- Funds: margin + fee escrowed from `available_base` into `order_locked_base`;
  `reward_gas` lamports transferred into the order PDA.
- Events: `OrderCreated` + `TriggerConditionCreated` (order.rs:670/690).

### 94 — MakeCloseOrderV2

Args `MakeCloseOrderV2Args` (instruction.rs:133): same fields as tag 93.

Accounts: same as tag 93 **minus the user-leverage account** (11 accounts).

- Pause gate: protocol/settlement → `V2Paused`; market OFFLINE → `TradingPaused`
  (closes allowed while PAUSED).
- Checks: position exists → `V2PositionNotFound` (439);
  `amount > size − freeze` → `V2CloseExceedsAvailable` (440), then `freeze_e9 += amount`;
  seq/reward/switch checks as tag 93 (no min-amount check, no leverage).
- Events: `OrderCreated` + `TriggerConditionCreated`.

### 95 — MakeStopTakeOrderV2

Args `MakeStopTakeOrderV2Args` (instruction.rs:162): `pair_id`, `direction`,
`amount_e9`, `reward_gas`, `order_seq`, `good_till`, `deadline`, plus
`variant: StopTakeVariantV2` (instruction.rs:146):
- `Open { trigger_price_e9, open_limit_price_e9, gain_trigger_price_e9, loss_trigger_price_e9 }`
- `Close { gain_trigger_price_e9, gain_limit_price_e9, loss_trigger_price_e9, loss_limit_price_e9 }`

Accounts: same 12 as tag 93 (the user-leverage PDA is passed on the Close variant too
but only key-checked).

- Switch bit: stop-take (`0b100`). Open variant: market not ACTIVE or trigger price 0
  → `V2TriggerNotMet` (435); limit-open chained with TP/SL →
  `V2UnsupportedConditionalChain` (451); funds escrowed like tag 93. Close variant:
  market OFFLINE or both triggers 0 → `V2TriggerNotMet`; position checks as tag 94 but
  **freeze is deferred to execution**.
- Events: `OrderCreated` + `TriggerConditionCreated` (trigger record `mean = 1`).

### 96 — TriggerOpenOrderV2 (keeper)

Args `TriggerOpenOrderV2Args` (instruction.rs:174): `order_seq: u64`, `page_id: u16`,
`maker_slot: u8`.

| # | Account | Flags |
| --- | --- | --- |
| 0 | keeper | [signer][writable] | pays deal-PDA rent |
| 1 | taker | [writable] | receives order/trigger rent |
| 2 | settlement mint | |
| 3 | protocol PDA | |
| 4 | settlement PDA | [writable] | fee accrual |
| 5 | market catalog PDA (page 0) | |
| 6 | market-order-config PDA | |
| 7 | trader PDA (taker) | [writable] |
| 8 | liquidity page PDA (args.page_id) | [writable] |
| 9 | market-risk PDA (order pair) | [writable] |
| 10 | sequence PDA | [writable] |
| 11 | deal PDA `["deal", mint, 0xC000…\|order_seq]` | [writable] | created fresh |
| 12 | order PDA | [writable] | closed on fill |
| 13 | trigger PDA | [writable] | closed on fill |
| 14 | oracle price account | | Custos/PPRC feed of the pair |
| 15 | system program | |
| 16–22 | **fee bundle** (7 accounts, fixed order) | see below |

**Fee bundle** (order.rs:445-533; exactly 7, else `V2MissingFeeAccounts` 407):

1. fee-recipients PDA `["fee-recipients"]` (must exist)
2. taker invite-relation PDA `["invite", taker]` (may be an empty system account)
3. direct commission PDA `["commission", mint, inviter‖taker]` [writable if present]
4. direct top-agent PDA `["top-agent", mint, inviter‖taker]`
5. upline invite-relation PDA `["invite", inviter‖taker]`
6. upline commission PDA `["commission", mint, upline]` [writable if present]
7. upline top-agent PDA `["top-agent", mint, upline]`

- Permission: **keeper** (`NotAuthorizedKeeper`). Pause gate: protocol/settlement → `V2Paused`.
- Checks: kind ∈ {limit-open, market-open} → `V2OrderKindMismatch` (437); market
  ACTIVE + oracle parity → `V2AccountMismatch`; limit price not crossed →
  `V2TriggerNotMet`; market deviation > config → `V2PriceDeviationTooLarge` (436);
  trader ACTIVE → `V2TraderLiquidationLocked`; margin rescale to 0 →
  `V2MarginTooSmall` (410); maker slot invalid/rejecting/frozen →
  `V2InvalidMakerSlot` (444); maker funds → `V2InsufficientMakerAvailable` (445);
  50 deals/position → `V2DealsPerPositionFull` (443).
- Flow: rescale frozen margin/fee to the fill price (shortfall topped from
  `available_base`, insufficient ⇒ cancel) → create deal → accrue fee synchronously
  (platform/trade/inviter/top-agent split) → pay `reward_gas` to keeper, order/trigger
  rent to taker.
- Events: `DealOpened` + `FeeAccrued` + `OrderTerminated state=3`.

### 97 — TriggerCloseOrderV2 (keeper)

Args `TriggerCloseOrderV2Args` (instruction.rs:181): `order_seq: u64`, `deal_id: u64`.

Accounts: keeper [signer][writable], taker [writable], mint, `protocol` PDA,
settlement PDA [writable], trader PDA [writable], deal PDA [writable],
liquidity page PDA [writable], market-risk PDA [writable], sequence PDA [writable],
order PDA [writable], trigger PDA [writable], oracle price account, then the
7-account fee bundle. **No system program** (no account creation).

- Permission: **keeper**. **No pause gate** (closes stay live while paused).
- Checks: kind ∈ {limit-close, market-close} → `V2OrderKindMismatch`; deal
  active/bound to taker+pair+direction → `V2DealPageMismatch` (446); limit price not
  crossed → `V2TriggerNotMet`; close amount 0 → `V2CloseExceedsRemaining` (447).
- Flow: `close = min(amount, deal.remaining)` → settle the single deal (pro-rata
  margin split; public-pool gain shortfall falls back to risk fund then bad debt) →
  close fee = notional × snap fee rate, **capped at trader equity**, accrued via the
  fee bundle → full close completes the order (reward → keeper, rent → taker);
  partial close leaves the order `PARTIAL`. Terminal deal PDAs may be reclaimed
  (rent → keeper) when the deal-reclaim feature flag is on.
- Events: `DealClosed`; `OrderTerminated state=3` on full close.

### 98 — TriggerStopTakeOpenV2 (keeper)

Args: `TriggerOpenOrderV2Args`. Accounts: same as tag 96 **without** the
market-order-config account (no deviation check).

- Permission: keeper. Pause gate: protocol/settlement → `V2Paused`.
- Checks: kind = stop-take-open → `V2OrderKindMismatch`; trigger price not crossed
  (long: mark ≥ trigger; short: mark ≤ trigger) → `V2TriggerNotMet`.
- Limit branch (`open_limit_price_e9 > 0`): lock funds at the limit price,
  insufficient ⇒ cancel (reward+rent → taker); otherwise pay half reward to keeper and
  convert to a limit-open order (`OrderConverted` b).
- Market branch: fill at mark like tag 96; if TP/SL legs are attached, pay half
  reward, freeze the position, and convert the same order into a stop-take-close
  (`good_till = now + ~10 years`, `OrderConverted` c, `event_index=2`); otherwise
  complete normally.
- Events: `DealOpened` + `FeeAccrued`; `OrderConverted`; `OrderTerminated`.

### 99 — TriggerStopTakeCloseV2 (keeper)

Args: `TriggerCloseOrderV2Args`. Accounts: same as tag 97.

- Permission: keeper. **No pause gate.**
- Checks: kind = stop-take-close → `V2OrderKindMismatch`; neither TP nor SL leg hit →
  `V2TriggerNotMet`. Mislabeled gain/loss legs are auto-swapped relative to entry
  (`adjust_close_params`, order.rs:58-84).
- Hit leg with a **limit price**: freeze the amount, pay half reward to keeper,
  convert to limit-close (`OrderConverted` a). Hit leg without limit price: market
  close through the tag-97 path.

### 100 — CancelOrderV2

Args `OrderSeqV2Args` (instruction.rs:187): `order_seq: u64`.

Accounts: `taker` [signer], mint, trader PDA [writable], order PDA [writable, closed],
trigger PDA [writable, closed].

- Permission: taker. **No pause gate, no switch check** — cancellation is always
  available. Liquidation-locked trader → `V2TraderLiquidationLocked` (use tag 104).
- Checks: order open & owned & trigger bound → `V2OrderNotPending` (429).
- Effect: reservation released (margin+fee back to available / position unfreeze),
  `reward_gas` refunded to taker, rent to taker.
- Event: `OrderTerminated state=4` (variant A, no `slot=`).

### 101 — ExpireOrderV2 (keeper)

Args: `OrderSeqV2Args`.

Accounts: `keeper` [signer], taker, mint, `protocol` PDA, trader PDA [writable],
order PDA [closed], trigger PDA [closed].

- Permission: **keeper**. Checks: still-open order past `good_till` required,
  otherwise `V2OrderExpired` (430).
- Effect: reservation released; reward → keeper; rent → taker.
- Event: `OrderTerminated state=6` (variant A).

### 102 — CleanupOrphanOrdersV2 (keeper)

Args `CleanupOrphanOrdersV2Args` (instruction.rs:192): `order_seqs: Vec<u64>`
(1–32 entries → else `InvalidArgument`).

Accounts: `keeper` [signer], taker, mint, `protocol` PDA, trader PDA [writable], then
`(order PDA, trigger PDA)` per seq; trailing extras → `InvalidArgument`.

- Permission: **keeper**. Trader must be ACTIVE → `V2TraderLiquidationLocked`.
- Orphan criteria per order: owned by taker, open, close-offset, kind ∈
  {limit-close, market-close, stop-take-close}, and the matching active position has
  `size_e9 == 0` — else `V2OrderNotOrphan` (448).
- Effect: reward → keeper, rent → taker (no reservation release).
- Event: `OrderTerminated reason=orphan state=4` (variant C, with `slot=`).

---

## Liquidation & Risk Fund

### 103 — LiquidateAccountBulkV2 (keeper, atomic packet)

Args `LiquidateAccountBulkV2Args` (instruction.rs:203): `expected_deal_count: u16`
(1–32), `expected_margin_nonce: u64` (= `trader.liquidation_nonce`),
`receipt_nonce: u64` (= nonce + 1, selects the receipt PDA),
`page_groups: Vec<PageGroupV2 { page_id: u16, deal_count: u16 }>` (strictly
increasing page ids; counts sum to `expected_deal_count`).

Fixed accounts: `keeper` [signer][writable], taker, mint, `protocol` PDA,
settlement PDA [writable], catalog page 0, trader PDA [writable], market-risk PDAs
for pairs 1/2/3 [writable], liquidation-receipt PDA
`["liq-receipt", mint, taker, receipt_nonce]` [writable, create-or-reset],
sequence PDA [writable], system program.

Dynamic tail (exact count enforced → `V2IncompleteDealTail` 457):

1. one oracle price account per unique active pair (first-occurrence order);
2. one liquidity page PDA [writable] per `page_groups` entry, in group order;
3. `expected_deal_count` deal PDAs [writable], consumed in page-group order.

- Permission: **keeper**; keeper may not liquidate itself (`Unauthorized`).
  **No pause gate.**
- Checks: trader ACTIVE|LIQUIDATING → `V2TraderLiquidationLocked`; nonce mismatch →
  `V2StaleNonce` (405); receipt belongs to other trader or already closed →
  `V2ReceiptReplay` (406); first packet requires at-risk account
  (`net_loss + maintenance ≥ equity`, MMR = settlement override ‖ protocol) →
  `V2NotLiquidatable` (454); > 32 deals → `V2BulkDealCapExceeded` (455); duplicate
  deal → `V2DuplicateDeal` (456); bad page groups → `V2InvalidPageGroups` (458).
- Per deal: full close at mark, **no close trading fee**; taker gain covered by maker
  equity → risk fund → bad debt; taker loss capped at equity, backstop risk fund →
  public pool. Outcomes appended to the receipt (≤ 32 → `V2ReceiptFull` 459).
- Finalize: when all deals are done, positions are cleared, **residual equity is swept
  to the risk fund of the lowest active pair**, status → LIQUIDATED; otherwise
  LIQUIDATING. Deal PDAs reclaimed (rent → keeper) when the deal-reclaim flag is on;
  receipt PDA closed (rent → keeper) when the ephemeral-close flag is on.
- Events: `DealLiquidated` (per deal), `SurplusToRisk`, `BulkLiquidation`,
  `LiquidationBadDebtItem`, `LiquidationReceiptAudit`, `DealRentReclaimed(Batch)`,
  `LiquidationReceiptRentReclaimed`.

### 104 — LiquidateCancelOrdersV2 (keeper)

Args `LiquidateCancelOrdersV2Args` (instruction.rs:215): `order_seqs: Vec<u64>`
(1–32 → else `InvalidArgument`).

Accounts: `keeper` [signer][writable], taker [writable], mint, `protocol` PDA,
trader PDA [writable], then `(order PDA, trigger PDA)` per seq (tail = 2×count →
`InvalidArgument`).

- Permission: **keeper**. **No pause gate, no nonce.**
- Checks: trader must be LIQUIDATED → `V2TraderNotLiquidated` (460); order open &
  owned → `V2OrderNotPending`. No fund refund or unfreeze (ledgers were zeroed by
  tag 103); escrowed reward → keeper, rent → taker.
- Event: `OrderTerminated reason=liquidation state=5` (variant D, per order).

### 105 — LiquidateMakerDealsBulkV2 (keeper)

Args `LiquidateMakerDealsBulkV2Args` (instruction.rs:220): `page_id: u16`,
`pair_id: u16` (1–3), `deal_ids: Vec<u64>` (1–4, strictly increasing →
`V2BulkDealCapExceeded`/`V2DuplicateDeal`).

Accounts: `keeper` [signer][writable], mint, `protocol` PDA, settlement PDA,
catalog page 0, liquidity page PDA [writable], market-risk PDA [writable],
oracle price account, sequence PDA [writable], then **(trader PDA [writable],
deal PDA [writable]) per deal_id** (tail = 2×count → `InvalidArgument`).

- Permission: **keeper**. **No pause gate.**
- Per deal: must be a private-pool deal (`InvalidPoolState`); deal on the public slot
  → `V2PrivateOnPublicSlot`; taker mid-liquidation ⇒ skipped. Trigger condition:
  `taker_gain ≥ maker_margin`. Auto-add (maker opted in, not frozen, sufficient
  available): top-up capped so `margin_after > taker_gain`, moving maker available →
  locked (`MakerMarginAdded source=auto`). Otherwise force-close: maker forfeits
  margin+maintenance (+available if needed), surplus → risk fund; shortfall paid by
  risk fund then bad debt; taker receives margin + capped `credited_gain`
  (`execution_price` recomputed). FROZEN flag clears when `open_deal_count` hits 0.
- Events: `MakerMarginAdded source=auto`, `MakerDealLiquidated`,
  `MakerLiquidationBatch`, `DealRentReclaimedBatch reason=3`.

### 106 — DepositRiskFundV2 / 107 — WithdrawRiskFundV2 (admin)

Args `RiskFundV2Args` (instruction.rs:228): `pair_id: u16` (1–3), `amount_base: u64`.

Accounts: `admin` [signer][writable], mint, `protocol` PDA, settlement PDA,
market-risk PDA [writable], admin token account [writable], custody PDA [writable],
custody-authority PDA, SPL token program.

- Permission: **admin**. Pause gate: none.
- Checks: pair 1–3 → `V2InvalidPair`; `amount > 0` → `InvalidAmount`; withdraw above
  `risk_fund_equity_base` → `V2InsufficientRiskFund` (461).
- SPL: deposit = admin → custody; withdraw = custody → admin (custody-authority signed).
- Events: `RiskFundDeposited` / `RiskFundWithdrawn` (liquidation.rs:1697).

### 108 — SetProtocolPausedV2

Args `SetProtocolPausedV2Args`: `paused: bool`.

Accounts: `admin` [signer], `protocol` PDA [writable].

- Permission: **admin**. Pause gate: none; this instruction is the pause/resume control.
- Event: `ProtocolPausedUpdated` with the old and new values.

### 109 — SetSettlementStatusV2

Args `SetSettlementStatusV2Args`: `status: u8` (`0 = ACTIVE`, `1 = PAUSED`).

Accounts: `admin` [signer], `protocol` PDA, settlement mint, settlement PDA [writable].

- Permission: **admin**. Pause gate: none.
- Checks: only ACTIVE and PAUSED are accepted; mint and settlement PDA must match.
- Event: `SettlementStatusUpdated` with the old and new status.

### 110 — UpdateSettlementConfigV2

Args `UpdateSettlementConfigV2Args` is a partial update. Each field is Borsh
`Option<T>`: `trading_fee_to_platform_e9`, `trading_fee_to_inviter_e9`,
`trading_fee_to_top_agent_e9`, `maintenance_margin_rate_e9`, and
`min_deposit_amount`.

Accounts: `admin` [signer], `protocol` PDA, settlement mint, settlement PDA [writable].

- Permission: **admin**. Pause gate: none.
- Checks: the three fee rates must sum to at most `1e9`; an explicitly supplied
  settlement maintenance rate must be in `(0, 1e9]`; mint and settlement PDA must match.
- Event: `SettlementConfigUpdated` with every old/new value.
