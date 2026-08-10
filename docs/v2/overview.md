---
id: v2-overview
title: V2 Overview & Compatibility
sidebar_position: 1
---

# Domi V2 Overview & Compatibility

This section documents the **Domi V2 parity lane**: instruction tags **60–107**, the
`domi_v2` PDA namespace, V2 account layouts, V2 error codes (400–461) and V2 log events.

Everything here is transcribed from the on-chain source at
`Fufuture-Contract` branch `domi_v2_dev` (HEAD `11ed56a`, `src/v2/`). If this document
and the source ever disagree, **the source wins**.

---

## Relationship to the Legacy Lane

- **Same Program ID.** V2 instructions are dispatched from the same
  `PerpetualInstruction` enum and the same program binary as the legacy lane
  (`src/processor/mod.rs:894-1109`). There is no separate V2 deployment and no
  separate Program ID. The placeholder `declare_id!` in `src/lib.rs:34` is shared by
  both lanes.
- **Legacy tags are unchanged.** Tags 0–59 keep their legacy meaning
  (e.g. `TradeFutures` = 7, `TriggerOrders` = 12, `LiquidateMakerDeals` = 17,
  `UpdateTakerBalances` = 42, `DistributeFees` = 52;
  `src/instructions/mod.rs:1519-1588`). V2 is append-only starting at tag 60
  (`src/instructions/mod.rs:1264`, `src/instructions/mod.rs:1594-1641`).
- **Account layouts are NOT compatible.** Legacy accounts and V2 accounts live in
  different PDA namespaces (`domi_v2`, `src/v2/pda.rs:8`) with different layouts.
  A V2 account carries a 4-byte header `version / is_initialized / bump / account_kind`
  and is Borsh-serialized with a fixed allocation `LEN` (`src/v2/state.rs:29-74`).
  Never pass a legacy account to a V2 instruction or vice versa.
- **Same oracle.** V2 reads prices through the Custos trusted-relayer Pyth receiver
  (PPRC-tagged `PriceUpdateV2` accounts, `src/oracle/pyth.rs:38-48`). The policy is
  unchanged: `ORACLE_POLICY_DOMI_CUSTOS_V1 = 1`, max staleness **150 slots**, max
  confidence **200 bps** (`src/v2/constants.rs:99-101`). Raw Pyth prices are normalized
  to E6 (`raw_price_to_e6`, `src/v2/funds.rs:144-169`) and then scaled **E6 × 1000 → E9**
  (`oracle_e6_to_e9`, `src/v2/math.rs:46-47`).

---

## Unit Conventions

All V2 fields and instruction arguments carry explicit unit suffixes:

| Suffix | Meaning | Example |
| --- | --- | --- |
| `_e9` | 1e9 fixed-point scale (`E9 = 1_000_000_000`, `src/v2/constants.rs:8`) | `leverage_e9 = 100e9` means 100× |
| `_base` | Settlement-mint base units (native token decimals) | `amount_base`, `equity_base` |
| `lamports` / `reward_gas` | Native SOL lamports | `reward_gas`, `minimum_reward_gas` |
| `_secs` / timestamps | Unix seconds (`i64`) | `good_till`, `deadline` |
| slots | Domichain slots (`u64`) | oracle staleness window |

## V2 Defaults and Capacities (frozen constants)

From `src/v2/constants.rs`:

| Constant | Value | Anchor |
| --- | --- | --- |
| Default market leverage | 100e9 | `DEFAULT_MARKET_LEVERAGE_E9`, constants.rs:83 |
| MMR | per-protocol config; settlement `maintenance_margin_rate_e9 = 0` means "inherit protocol" | `effective_mmr_e9`, state.rs:280-286 |
| Min order amount | 1 (E9) | `DEFAULT_MIN_ORDER_AMOUNT_E9`, constants.rs:84 |
| Max market-open deviation | 5% (50_000_000 E9) | `DEFAULT_MAX_MARKET_OPEN_DEVIATION_E9`, constants.rs:85 |
| Min keeper reward | 3_000_000 lamports | `DEFAULT_MINIMUM_REWARD_GAS`, constants.rs:86 |
| Default order TTL | 7 days | `DEFAULT_ORDER_GOOD_TILL_SECS`, constants.rs:87 |
| TP/SL TTL (after conversion) | ~10 years | `TPSL_GOOD_TILL_SECS`, constants.rs:88 |
| Fast-lane pairs | 1–3 | `MIN/MAX_FAST_LANE_PAIR`, constants.rs:23-24 |
| Pending orders / trader | 32 | `MAX_PENDING_ORDERS`, constants.rs:16 |
| Position slots / trader | 8 | `MAX_POSITION_SLOTS`, constants.rs:14 |
| Deals / position | 50 | `MAX_DEALS_PER_POSITION`, constants.rs:15 |
| Makers / page | 16 | `MAKERS_PER_PAGE`, constants.rs:17 |
| Deal slots / page | 48 | `DEAL_SLOTS_PER_PAGE`, constants.rs:18 |
| Taker bulk-liquidation packet | ≤ 32 deals | `MAX_BULK_DEALS`, constants.rs:19 |
| Maker-liquidation packet | ≤ 4 deals | `MAX_MAKER_LIQUIDATION_DEALS`, constants.rs:20 |
| Keepers | ≤ 16 | `MAX_KEEPERS`, constants.rs:12 |
| Receipt outcomes | ≤ 32 | `MAX_RECEIPT_OUTCOMES`, constants.rs:21 |

## Reading the Documents

- [V2 Instructions](./instructions.md) — tags 60–107, arguments, accounts, permissions, pause gates.
- [V2 PDAs & Account Layouts](./pda.md) — all `domi_v2` seeds and Borsh layouts.
- [V2 Events](./events.md) — every `DOMI_V2_EVENT` log line, for frontends and indexers.
- [V2 Error Codes](./errors.md) — codes 400–461.
- [Legacy ↔ V2 Flow Comparison & Parameter Manifest](./flows.md) — transaction-flow parity and an integration checklist.
