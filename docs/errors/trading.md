---
id: errors-trading-validation
title: Trading & Validation Errors (350–379)
sidebar_position: 15
---

# Trading & Validation Errors (350–379)

These errors occur during trade execution, trigger validation, escrow checks, and final settlement validation.

---

# Error Table

| Code | Name | Message | Cause | Solution |
|------|------------------------------|------------------------------|----------------------------------------------|----------------------------------------------|
| 350 | MissingAssetAccount | Missing asset account | Required asset account missing | Provide required asset accounts |
| 351 | EscrowAccountUnderwater | Escrow account underwater | Escrow balance negative | Add funds to escrow |
| 352 | TradingNotAllowed | Trading not allowed | Trading disabled | Verify permissions |
| 353 | InvalidPNLCalculation | Invalid PNL calculation | PNL computation error | Report calculation issue |
| 354 | NothingToLiquidate | Nothing to liquidate | Position healthy | Cannot liquidate |
| 355 | InvalidOrderId | Invalid order ID | ID format invalid | Use valid u64 ID |
| 356 | InvalidDirection | Invalid direction | Not 0 (LONG) or 1 (SHORT) | Use valid direction |
| 357 | InvalidTriggerType | Invalid trigger type | Trigger type unknown | Use supported type |
| 358 | InvalidTriggerPrices | Invalid trigger prices | Stop/take prices invalid | Set valid trigger levels |
| 359 | InvalidLimitPrice | Invalid limit price | Price zero or unreasonable | Use positive limit price |
| 360 | TriggerNotMet | Trigger not met | Market not reached level | Wait for trigger |
| 361 | TradingPaused | Trading paused | `Config.isPaused = true` | Wait for unpause |
| 362 | TransactionExpired | Transaction expired | `goodTill` passed | Increase expiry |
| 363 | InsufficientReward | Insufficient reward | Reward too low | Increase reward gas |
| 364 | InvalidUserAccount | Invalid user account | Account uninitialized | Initialize user account |

---

# Characteristics

- Triggered during final trade validation  
- Enforce trigger correctness  
- Validate direction and order identifiers  
- Protect escrow solvency  
- Enforce pause and expiry logic  
