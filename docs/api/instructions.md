---
id: instruction-discriminators
title: Instruction Discriminators
sidebar_position: 2
---

# Instruction Discriminators

Every instruction in the `PerpetualInstruction` enum is encoded using **Borsh**.

The **first byte** of serialized data is the **enum variant index**, also called the **instruction discriminator**.

This byte determines which instruction the program executes.

---

# How It Works

When serialized:

```
<u8 discriminator> + <borsh-encoded fields>
```

Example:

```ts
Buffer.concat([
  Buffer.from([7]),   // TradeFutures discriminator
  ...encodedFields
])
```

The Rust side resolves it via:

```rust
pub fn discriminator(&self) -> u8 {
    match self {
        Self::Deposit { .. } => 0,
        ...
    }
}
```

---

# Complete Discriminator Table

## User Balance Management (0–2)

| Discriminator | Instruction |
|--------------|-------------|
| 0 | Deposit |
| 1 | Withdraw |
| 2 | WithdrawTo |

---

## Order Management (3–8)

| Discriminator | Instruction |
|--------------|-------------|
| 3 | OpenLimit |
| 4 | CloseLimit |
| 5 | CancelLimit |
| 6 | MatchOrder |
| 7 | TradeFutures |
| 8 | ClosePosition |

---

## Conditional Orders (9–11)

| Discriminator | Instruction |
|--------------|-------------|
| 9 | PlaceConditionalOrder |
| 10 | CancelConditionalOrder |
| 11 | PlaceBracketOrder |

---

## Triggers & Execution (12–14)

| Discriminator | Instruction |
|--------------|-------------|
| 12 | TriggerOrders |
| 13 | TriggerOpenOrder |
| 14 | TriggerCloseOrder |

---

## Liquidation (15–18)

| Discriminator | Instruction |
|--------------|-------------|
| 15 | ClearingTaker |
| 16 | LiquidateMakerDeal |
| 17 | LiquidateMakerDeals |
| 18 | AfterLiquidateTaker |

---

## Public Pool Operations (19–21)

| Discriminator | Instruction |
|--------------|-------------|
| 19 | InitializePublicPool |
| 20 | AddPublicLiquidity |
| 21 | WithdrawPublicLiquidity |

---

## Private Pool Operations (22–24)

| Discriminator | Instruction |
|--------------|-------------|
| 22 | InitializePrivatePool |
| 23 | AddPrivateLiquidity |
| 24 | WithdrawPrivateLiquidity |

---

## Pool Management (25–27)

| Discriminator | Instruction |
|--------------|-------------|
| 25 | MoveToPublic |
| 26 | AddMarginAmount |
| 27 | SetUserParam |

---

## Pool CPI (Internal) (28–29)

| Discriminator | Instruction |
|--------------|-------------|
| 28 | PoolMatchOrder |
| 29 | PoolCloseOrder |

---

## Admin Functions (30–39)

| Discriminator | Instruction |
|--------------|-------------|
| 30 | SetUnderlying |
| 31 | SetKeepers |
| 32 | SetTradeFeeAddr |
| 33 | SetMaintenanceMarginRate |
| 34 | SetTradeFeeRate |
| 35 | SetLimitFeeRate |
| 36 | SetStartStop |
| 37 | SetPoolPublishAppendRate |
| 38 | InitializeConfig |
| 39 | InitializeUserAccount |
| 40 | InitializePoolList |

---

## View Functions (42–47)

| Discriminator | Instruction |
|--------------|-------------|
| 43 | GetPosition |
| 44 | GetOrder |
| 45 | GetLimitedOrder |
| 46 | GetPNLByCertainPosition |
| 47 | CheckTakerRiskStatus |

---

# Important Notes

- Discriminators **must match Rust enum ordering exactly**.
- Changing enum order will break compatibility.
- Always hardcode discriminator values in frontend builders.
- Never rely on implicit enum ordering in TypeScript.

---

# Example: Manual Encoding

```ts
const data = Buffer.concat([
  Buffer.from([0]), // Deposit
  writeU128LE("1000000000000000000")
]);
```

```ts
const data = Buffer.concat([
  Buffer.from([7]), // TradeFutures
  ...encodedFields
]);
```

---

# Validation Rule

Before decoding:

```rust
pub fn validate_instruction_data(data: &[u8]) -> Result<(), ProgramError> {
    if data.is_empty() {
        return Err(InstructionUnpackError.into());
    }
    Ok(())
}
```

The first byte determines the variant.

---

This table is the canonical reference for all client-side instruction builders.
