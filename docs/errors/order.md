---
id: errors-order
title: Order Errors (40–79)
sidebar_position: 4
---

# Order Errors (40–79)

These errors relate to order lifecycle, trading pair validation, and order parameter constraints.

---

# Error Table

| Code | Name | Message | Cause | Solution |
|------|----------------------|--------------------------|---------------------------------------------|----------------------------------------------|
| 40 | OrderNotFound | Order not found | Deal ID does not exist | Verify order ID |
| 41 | InvalidOrderState | Invalid order state | Order state does not allow operation | Ensure correct state (ACTIVE, PENDING, etc.) |
| 42 | OrderNotActive | Order not active | Order closed, cancelled, or liquidated | Cannot operate on inactive order |
| 43 | OrderAlreadyClosed | Order already closed | Attempted to close closed order | Check state before closing |
| 44 | InvalidOrderAmount | Invalid order amount | Zero or exceeds limits | Use between `minOrderSize` and `maxOrderSize` |
| 45 | OrderTooSmall | Order too small | Below `Config.minOrderSize` | Increase order size |
| 46 | OrderTooLarge | Order too large | Exceeds `Config.maxOrderSize` | Reduce or split order |
| 47 | InvalidOrderName | Invalid order name | Invalid trading pair format | Use format like `BTC-USD` |
| 48 | OrderNameMismatch | Order name mismatch | Pair does not match expected | Verify trading pair |
| 49 | OrderStateMismatch | Order state mismatch | State conflicts with operation | Validate state compatibility |
| 50 | InvalidAssetName | Invalid asset name | Name invalid or >64 chars | Use valid name under 64 chars |
| 51 | InvalidOrderSize | Invalid order size | Order size parameter invalid | Provide positive value |
| 52 | InvalidAmount | Invalid amount | Amount zero or invalid | Use positive non-zero amount |
| 53 | AssetNotEnabled | Asset not enabled | Pair disabled in config | Enable asset before trading |
| 54 | AssetNotCollateral | Asset not collateral | Asset not allowed as collateral | Use approved collateral |
| 55 | AssetNotInitialized | Asset not initialized | Asset account missing | Call `SetUnderlying` first |
| 56 | InvalidPriceData | Invalid price data | Malformed or invalid price | Verify oracle/price feed |
| 57 | InvalidDealId | Invalid deal ID | Deal ID invalid or out of range | Use valid active deal ID |
| 58 | CloseFailed | Close operation failed | Position close encountered error | Verify position state |

---

# Characteristics

- Triggered during open/close/modify order flows
- Enforce pair configuration constraints
- Enforce order size bounds
- Validate asset state and enablement
- Dependent on correct deal and state tracking
