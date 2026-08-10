---
id: v2-pda
title: V2 PDAs & Account Layouts
sidebar_position: 3
---

# V2 PDAs & Account Layouts

All V2 state lives in Program Derived Addresses under the `domi_v2` namespace
(`SEED_NAMESPACE = b"domi_v2"`, `src/v2/pda.rs:8`). Derivations below are transcribed
from `src/v2/pda.rs`; layouts from `src/v2/state.rs`.

**Encoding rules** (`src/v2/pda.rs:1-4`):

- Every seed list starts with `b"domi_v2"`.
- `u16` seeds (`page_id`, `pair_id`) are **2-byte little-endian** (`to_le_bytes`).
- `u64` seeds (`order_seq`, `deal_id`, `nonce`) are **8-byte little-endian**.
- Pubkey seeds are the raw 32 bytes.

---

## PDA Seed Table

| Account | Seeds (after `domi_v2`) | Derivation fn |
| --- | --- | --- |
| ProtocolConfig | `["protocol"]` | `pda::protocol` (pda.rs:31) |
| SettlementConfig | `["settlement", mint]` | `pda::settlement` (pda.rs:35) |
| MarketCatalogPage | `["markets", mint, page_id u16le]` | `pda::market_catalog` (pda.rs:42) |
| MarketOrderConfig | `["order-market", mint, pair_id u16le]` | `pda::market_order_config` (pda.rs:54) |
| SequenceCounter | `["seq", mint]` | `pda::sequence` (pda.rs:66) |
| UserLeverage | `["ulev", mint, pair_id u16le, owner]` | `pda::user_leverage` (pda.rs:70) |
| TraderMargin | `["trader", mint, owner]` | `pda::trader` (pda.rs:88) |
| LiquidityPage | `["liquidity", mint, page_id u16le]` | `pda::liquidity_page` (pda.rs:95) |
| Custody (SPL token account) | `["custody", mint]` | `pda::custody` (pda.rs:107) |
| CustodyAuthority | `["custody-auth", mint]` | `pda::custody_authority` (pda.rs:111) |
| CommissionAccount | `["commission", mint, owner]` | `pda::commission` (pda.rs:118) |
| FeeRecipients | `["fee-recipients"]` | `pda::fee_recipients` (pda.rs:130) |
| InviteRelation | `["invite", invitee]` | `pda::invite_relation` (pda.rs:134) |
| TopAgent | `["top-agent", mint, agent]` | `pda::top_agent` (pda.rs:138) |
| PublicShare | `["pub-share", mint, holder]` | `pda::public_share` (pda.rs:150) |
| MarketTemplate | `["market-template"]` | `pda::market_template` (pda.rs:162) |
| OrderRecord | `["order", mint, order_seq u64le]` | `pda::order` (pda.rs:166) |
| TriggerCondition | `["trigger", mint, order_seq u64le]` | `pda::trigger` (pda.rs:178) |
| DealRecord | `["deal", mint, deal_id u64le]` | `pda::deal` (pda.rs:190) |
| MarketRisk | `["risk", mint, pair_id u16le]` | `pda::market_risk` (pda.rs:202) |
| LiquidationReceipt | `["liq-receipt", mint, trader, nonce u64le]` | `pda::liquidation_receipt` (pda.rs:214) |

Notes:

- `mint` is always the **settlement SPL mint**; every settlement-scoped PDA includes it.
- The **custody** PDA is an SPL token account (owner = SPL token program) whose token
  authority is the **custody-authority** PDA; the program signs payouts with
  `invoke_signed` over `["domi_v2", "custody-auth", mint, bump]`
  (`transfer_from_custody_checked`, `src/v2/funds.rs:530-561`).
- Trigger-created deals use `deal_id = 0xC000_0000_0000_0000 | order_seq`
  (`trigger_deal_id`, `src/v2/constants.rs:103-104,136-141`), so order-derived deal
  PDAs never collide with sequence-allocated deal ids.

---

## Account Serialization Model

Transcribed from `src/v2/state.rs:29-74`:

- **Borsh** serialization (derive on every struct); all integers little-endian,
  `bool` = 1 byte, `Pubkey` = 32 bytes, `Vec<T>` = 4-byte u32 count prefix + payload.
- Every account starts with a **4-byte header**:
  `version: u8` | `is_initialized: bool` | `bump: u8` | `account_kind: u8`.
- `LEN` is the fixed on-chain allocation, sized for worst-case `Vec` content;
  `pack` zero-fills the tail, so readers must Borsh-deserialize from the front and
  ignore trailing zeros. Some accounts carry deliberate slack (noted below).
- `unpack` rejects wrong length (`InvalidAccountData`), `version != 1`
  (`V2InvalidVersion` 400) and wrong kind/uninitialized (`V2InvalidAccountKind` 413).

`account_kind` values (`src/v2/constants.rs:106-124`):

| kind | Account | kind | Account |
| --- | --- | --- | --- |
| 1 | ProtocolConfigV2 | 11 | InviteRelationV2 |
| 2 | SettlementConfigV2 | 12 | TopAgentV2 |
| 3 | MarketCatalogPageV2 | 13 | PublicShareV2 |
| 4 | MarketOrderConfigV2 | 14 | MarketTemplatePageV2 |
| 5 | SequenceCounterV2 | 15 | OrderRecordV2 |
| 6 | UserLeverageV2 | 16 | TriggerConditionV2 |
| 7 | TraderMarginV2 | 17 | DealRecordV2 |
| 8 | LiquidityPageV2 | 18 | MarketRiskPageV2 |
| 9 | CommissionAccountV2 | 19 | LiquidationReceiptV2 |
| 10 | FeeRecipientsV2 | | |

---

## Account Layouts

All field tables list fields **in Borsh declaration order**. The 4-byte header is
omitted from every table (always present, always first).

### ProtocolConfigV2 — kind 1, LEN 613 (state.rs:91-205)

| Field | Type | Notes |
| --- | --- | --- |
| admin | Pubkey | protocol admin |
| maintenance_margin_rate_e9 | u64 | protocol MMR, must be > 0 |
| min_deposit_amount | u64 | base units; settlement value overrides when non-zero |
| paused | bool | global pause gate |
| next_deal_id | u64 | starts at 1 |
| keepers | Vec\<Pubkey\> | ≤ 16 (`MAX_KEEPERS`) |
| reserved | Vec\<u8\> | ≤ 32; `[0]` = order-switch bitmask (0/unset ⇒ `0b111` all on), `[1]` = permissionless flag |

### SettlementConfigV2 — kind 2, LEN 208 (state.rs:208-330)

| Field | Type | Notes |
| --- | --- | --- |
| settlement_mint | Pubkey | |
| custody | Pubkey | custody token account |
| custody_authority_bump | u8 | |
| decimals | u8 | mint decimals |
| status | u8 | 0 = ACTIVE, 1 = PAUSED |
| min_deposit_amount | u64 | base units |
| maintenance_margin_rate_e9 | u64 | **0 = inherit protocol MMR** |
| trading_fee_to_platform_e9 | u64 | default 0.2e9 (`DEFAULT_FEE_PLATFORM_E9`) |
| trading_fee_to_inviter_e9 | u64 | default 0.1e9 |
| trading_fee_to_top_agent_e9 | u64 | default 0.5e9 |
| platform_fee_accrued_base | u64 | claimable by `platform_owner` |
| trade_fee_accrued_base | u64 | claimable by `trade_fee_owner` |
| commission_accrued_base | u64 | invariant: 0 after fee accrual |
| liquidity_page_count | u16 | |
| public_page_id | u16 | `u16::MAX` = public pool not configured |
| public_maker_slot | u8 | `u8::MAX` when unconfigured |
| total_public_shares | u64 | |
| public_pool_equity_base | u64 | |
| public_available_base | u64 | |
| public_locked_base | u64 | |
| reserved | Vec\<u8\> | ≤ 32; `[0]` = lifecycle flags, default `0b111` (deal-id-v2, deal-reclaim, receipt-ephemeral-close) |

### MarketEntryV2 — embedded, MAX 97 bytes (state.rs:333-375)

| Field | Type | Notes |
| --- | --- | --- |
| pair_id | u16 | 1–3 |
| active | bool | |
| status | u8 | 0 = ACTIVE, 1 = PAUSED, 2 = OFFLINE |
| oracle_policy_version | u8 | must be 1 (`ORACLE_POLICY_DOMI_CUSTOS_V1`) |
| oracle_feed_id | [u8; 32] | Custos/PPRC feed id |
| multi_e9 | u64 | price multiplier; 0 or exactly 1e9 |
| lot_multi | u64 | |
| trading_fee_rate_e9 | u64 | ≤ 1e9 |
| max_leverage_e9 | u64 | ≤ 1000e9 |
| min_leverage_e9 | u64 | ≥ 1e9 |
| name | String | ≤ 16 bytes |

### MarketCatalogPageV2 — kind 3, LEN 1630 (state.rs:378-435)

`settlement_mint: Pubkey`, `page_id: u16`, `markets: Vec<MarketEntryV2>` (≤ 16),
`reserved: Vec<u8>` (≤ 32).

### MarketTemplatePageV2 — kind 14, LEN 1630 (state.rs:438-494)

`markets: Vec<MarketEntryV2>` (≤ 16), `reserved: Vec<u8>` (≤ 32). LEN intentionally
matches MarketCatalogPageV2 (34 bytes slack).

### MarketOrderConfigV2 — kind 4, LEN 116 (state.rs:497-549)

| Field | Type | Notes |
| --- | --- | --- |
| settlement_mint | Pubkey | |
| pair_id | u16 | |
| default_leverage_e9 | u64 | default 100e9 |
| min_order_amount_e9 | u128 | default 1 |
| order_override_mask | u8 | per-kind override-enable bits (market 0b001 / limit 0b010 / stop-take 0b100) |
| order_override_value | u8 | per-kind on/off when mask bit set |
| max_market_open_deviation_e9 | u64 | default 0.05e9 (5%) |
| minimum_reward_gas | u64 | lamports, default 3_000_000 |
| reserved | Vec\<u8\> | ≤ 32 |

### SequenceCounterV2 — kind 5, LEN 128 (state.rs:626-687)

`settlement_mint: Pubkey`, `next_deal_seq: u64` (starts 1), `next_order_seq: u64`
(starts 0), `next_history_seq: u64`, `reserved: Vec<u8>` (≤ 64).

### UserLeverageV2 — kind 6, LEN 98 (state.rs:552-623)

`owner: Pubkey`, `settlement_mint: Pubkey`, `pair_id: u16`, `leverage_e9: u64`,
`reserved: Vec<u8>` (≤ 16).

### TraderMarginV2 — kind 7, LEN 4457 (state.rs:747-936)

| Field | Type | Notes |
| --- | --- | --- |
| owner | Pubkey | |
| settlement_mint | Pubkey | |
| status | u8 | 0 = ACTIVE, 1 = LIQUIDATING, 2 = LIQUIDATED |
| equity_base | u64 | |
| available_base | u64 | |
| margin_locked_base | u64 | |
| order_locked_base | u64 | margin+fee escrowed by open orders |
| liquidation_nonce | u64 | optimistic-concurrency counter for liquidation packets |
| last_time | i64 | |
| positions | Vec\<PositionSlotV2\> | ≤ 8 |
| pending_order_ids | Vec\<u64\> | ≤ 32 |
| reserved | Vec\<u8\> | ≤ 32 |

Conservation invariant while ACTIVE: `available + margin_locked + order_locked ≤ equity`
(state.rs:921-930).

### PositionSlotV2 — embedded, MAX 505 bytes (state.rs:690-744)

| Field | Type | Notes |
| --- | --- | --- |
| pair_id | u16 | |
| direction | u8 | 1 = long, 2 = short |
| active | bool | |
| size_e9 | u128 | |
| value_e9 | u128 | entry notional; avg price = value × E9 / size |
| margin_base | u64 | |
| freeze_e9 | u128 | frozen by close orders, ≤ size |
| oracle_feed_id | [u8; 32] | |
| oracle_policy_version | u8 | |
| multi_e9 | u64 | snapshot |
| deal_ids | Vec\<u64\> | ≤ 50; 0 = tombstone |

### LiquidityPageV2 — kind 8, LEN 4966 (state.rs:1622-1858)

| Field | Type | Notes |
| --- | --- | --- |
| settlement_mint | Pubkey | |
| page_id | u16 | |
| free_maker_bitmap | u32 | bit i = maker slot i free |
| free_deal_bitmap | [u64; 4] | 256 bits; bit i = deal slot i free |
| makers | Vec\<MakerSlotV2\> | exactly 16 entries |
| deal_slots | Vec\<LiquidityDealSlotV2\> | exactly 48 entries |
| reserved | Vec\<u8\> | ≤ 16 |

### MakerSlotV2 — embedded, MAX 94 bytes (state.rs:1514-1584)

| Field | Type | Notes |
| --- | --- | --- |
| occupied | bool | |
| holder | Pubkey | maker wallet; public slot holder = custody-authority PDA |
| equity_base / available_base / locked_base / maintenance_base | u64 ×4 | |
| leverage_e9 | u64 | default 10e9 |
| mmr_e9 | u64 | default 0.1e9 |
| add_margin_rate_e9 | u64 | default 0.1e9 |
| auto_add_margin | bool | legacy default true until params flag set |
| reject_order | bool | |
| flags | u8 | bit0 = FROZEN, bit1 = PARAMS_INITIALIZED |
| open_deal_count | u16 | |

### LiquidityDealSlotV2 — embedded, MAX 70 bytes (state.rs:1587-1619)

`occupied: bool`, `deal_id: u64`, `maker_slot: u8`, `state: u8` (DEAL_* values),
`pair_id: u16`, `direction: u8`, `amount_e9: u128`, `remaining_e9: u128`,
`entry_price_e9: u64`, `margin_base: u64`, `maintenance_base: u64`.

### CommissionAccountV2 — kind 9, LEN 128 (state.rs:939-997)

`settlement_mint: Pubkey`, `owner: Pubkey`, `accrued_base: u64`, `claimed_base: u64`,
`reserved: Vec<u8>` (≤ 16). Claimable = accrued − claimed. (24 B slack.)

### FeeRecipientsV2 — kind 10, LEN 128 (state.rs:1000-1052)

`platform_owner: Pubkey`, `trade_fee_owner: Pubkey`, `locked: bool` (set at init),
`reserved: Vec<u8>` (≤ 32).

### InviteRelationV2 — kind 11, LEN 128 (state.rs:1055-1087)

`invitee: Pubkey`, `inviter: Pubkey`, `created_at: i64`, `reserved: Vec<u8>` (≤ 16).

### TopAgentV2 — kind 12, LEN 128 (state.rs:1090-1121)

`settlement_mint: Pubkey`, `agent: Pubkey`, `approved: bool`, `updated_at: i64`,
`reserved: Vec<u8>` (≤ 16).

### PublicShareV2 — kind 13, LEN 128 (state.rs:1124-1167)

`holder: Pubkey`, `settlement_mint: Pubkey`, `shares: u64`, `last_time: i64`,
`reserved: Vec<u8>` (≤ 16). First public deposit mints 1 share per base unit.

### OrderRecordV2 — kind 15, LEN 272 (state.rs:1170-1283)

| Field | Type | Notes |
| --- | --- | --- |
| order_seq | u64 | |
| taker | Pubkey | |
| settlement_mint | Pubkey | |
| pair_id | u16 | |
| direction | u8 | 1 = long, 2 = short |
| offset | u8 | 1 = open, 2 = close |
| order_kind | u8 | 1 limit-open, 2 limit-close, 3 stop-take-open, 4 stop-take-close, 5 market-open, 6 market-close |
| state | u8 | 1 pending, 2 partial, 3 completed, 4 cancelled, 5 liquidated, 6 expired, 7 exception |
| amount_e9 | u128 | |
| target_price_e9 | u64 | |
| margin_base | u64 | escrowed (open orders) |
| trading_fee_base | u64 | escrowed (open orders) |
| reward_gas | u64 | lamports escrowed in the order PDA |
| start_time / good_till | i64 ×2 | unix seconds |
| oracle_feed_id | [u8; 32] | snapshot |
| oracle_policy_version | u8 | snapshot |
| snap_fee_rate_e9 | u64 | snapshot |
| snap_multi_e9 | u64 | snapshot |
| snap_leverage_e9 | u64 | snapshot; 0 on close orders |
| reserved | Vec\<u8\> | ≤ 64 |

### TriggerConditionV2 — kind 16, LEN 96 (state.rs:1286-1330)

`order_seq: u64`, `mean: u8` (0 invalid, 1 pure-condition, 2 execute-after,
3 cancelled, 4 completed), `open_limit_price_e9: u64`, `gain_trigger_price_e9: u64`,
`gain_limit_price_e9: u64`, `loss_trigger_price_e9: u64`, `loss_limit_price_e9: u64`,
`reserved: Vec<u8>` (≤ 32).

### DealRecordV2 — kind 17, LEN 256 (state.rs:1333-1440)

| Field | Type | Notes |
| --- | --- | --- |
| id | u64 | sequence deal id, or `0xC000… \| order_seq` for trigger fills |
| state | u8 | 1 active, 2 partial, 3 closed, 4 liquidated, 5 agreement (maker force-close) |
| taker | Pubkey | |
| settlement_mint | Pubkey | |
| pair_id | u16 | |
| direction | u8 | |
| amount_e9 / remaining_e9 | u128 ×2 | |
| entry_price_e9 | u64 | |
| taker_margin_base | u64 | |
| trading_fee_base | u64 | |
| position_slot | u8 | < 8 |
| position_deal_index | u8 | < 50 |
| liquidity_page_id | u16 | |
| maker_slot | u8 | < 16 |
| liquidity_deal_slot | u16 | < 48 |
| pool_kind | u8 | 1 = public, 2 = private |
| opened_at / closed_at | i64 ×2 | |
| liquidation_nonce | u64 | packet nonce (liquidated deals) |
| realized_gain_base / realized_loss_base | u64 ×2 | |
| reserved | Vec\<u8\> | ≤ 16 |

### MarketRiskPageV2 — kind 18, LEN 192 (state.rs:1443-1512)

`settlement_mint: Pubkey`, `pair_id: u16`, `public_pool_equity_base: u64`,
`risk_fund_equity_base: u64`, `long_oi_e9: u128`, `short_oi_e9: u128`,
`long_notional_e9: u128`, `short_notional_e9: u128`, `liquidations: u64`,
`bad_debt_absorbed_base: u64`, `reserved: Vec<u8>` (≤ 32).

### LiquidationReceiptV2 — kind 19, LEN 2007 (state.rs:1902-1979)

| Field | Type | Notes |
| --- | --- | --- |
| trader | Pubkey | |
| settlement_mint | Pubkey | |
| nonce | u64 | = `trader.liquidation_nonce` of the finalizing packet |
| deal_count | u16 | = outcomes.len() |
| pair_mask | u64 | bitmask of pair ids touched |
| timestamp | i64 | |
| closed | bool | set when the trader's last deal is liquidated |
| outcomes | Vec\<LiquidationOutcomeV2\> | ≤ 32 |
| reserved | Vec\<u8\> | ≤ 16 |

`LiquidationOutcomeV2` (embedded, 59-byte LE audit encoding, state.rs:1861-1897):
`deal_id: u64`, `pair_id: u16`, `direction: u8`, `amount_e9: u128`, `price_e9: u64`,
`gain_base: u64`, `loss_base: u64`, `bad_debt_base: u64`.
