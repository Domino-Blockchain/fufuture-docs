---
id: errors-configuration
title: Configuration Errors (200–219)
sidebar_position: 10
---

# Configuration Errors (200–219)

These errors relate to system configuration, keeper management, and trading pair setup.

---

# Error Table

| Code | Name | Message | Cause | Solution |
|------|------------------------------|-------------------------------|--------------------------------------------------|------------------------------------------------|
| 200 | TooManyKeepers | Too many keepers | Keeper list at max capacity | Remove unused keepers first |
| 201 | KeeperAlreadyExists | Keeper already exists | Keeper already authorized | Keeper already active |
| 202 | KeeperNotFound | Keeper not found | Keeper not in list | Cannot remove non-existent keeper |
| 203 | CannotRemoveLastKeeper | Cannot remove last keeper | Attempt to remove only keeper | At least one keeper required |
| 204 | NoKeepersConfigured | No keepers configured | No authorized keepers set | Add at least one keeper |
| 205 | InvalidParameter | Invalid parameter | Config value invalid | Use valid parameter range |
| 206 | InvalidAccountState | Invalid account state | State inconsistent/corrupted | Reinitialize if necessary |
| 207 | DepositAmountTooSmall | Deposit amount too small | Below `Config.minDepositAmount` | Increase deposit |
| 208 | TradingPairAlreadyExists | Trading pair already exists | Pair already enabled | Pair already configured |
| 209 | TooManyTradingPairs | Too many trading pairs | Max (20) reached | Remove unused pairs |
| 210 | TradingPairNotFound | Trading pair not found | Pair not configured | Add trading pair first |

---

# Characteristics

- Enforced during admin configuration flows  
- Protect keeper list integrity  
- Enforce trading pair limits  
- Validate system parameter bounds  
