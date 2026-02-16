---
id: errors-deal-state
title: Deal & State Errors (300–349)
sidebar_position: 14
---

# Deal & State Errors (300–349)

These errors relate to deal lifecycle, position state management, and trigger logic.

---

# Error Table

| Code | Name | Message | Cause | Solution |
|------|-------------------------------|-------------------------------|----------------------------------------------|----------------------------------------------|
| 300 | DealAlreadyExists | Deal already exists | Deal ID already used | Use next deal ID |
| 301 | DealNotFound | Deal not found | Deal missing or closed | Verify active deal ID |
| 302 | DealAlreadyClosed | Deal already closed | Operation on closed deal | Cannot modify closed deal |
| 303 | DealNotActive | Deal not active | State not ACTIVE | Operate only on active deals |
| 304 | InsufficientAmount | Insufficient amount | Size too small for operation | Reduce close amount |
| 305 | InvalidDealState | Invalid deal state | State invalid for action | Check required state |
| 306 | LimitOrderAlreadyClosed | Limit order already closed | Completed or cancelled | Cannot operate on closed order |
| 307 | LimitOrderNotFound | Limit order not found | Account missing | Verify limit order ID |
| 308 | InvalidStateTransition | Invalid state transition | State change not allowed | Follow valid transitions |
| 309 | NotOrderHolder | Not order holder | Signer not owner | Only holder can act |
| 310 | InvalidReturnData | Invalid return data | Return format incorrect | Verify return structure |
| 311 | InvalidReturnDataLength | Invalid return data length | Length mismatch | Check expected size |
| 312 | NoReturnData | No return data | Expected return missing | Ensure instruction returns data |
| 313 | PositionNotFound | Position not found | No position exists | Open position first |
| 314 | TriggerConditionNotFound | Trigger condition not found | Trigger missing | Create trigger first |
| 315 | TriggerConditionAlreadyExists | Trigger condition already exists | Trigger already set | Update instead |
| 316 | InvalidTriggerCondition | Invalid trigger condition | Trigger params invalid | Check trigger price/type |
| 317 | PositionSizeZero | Position size zero | No size remaining | Cannot operate |
| 318 | DealMismatch | Deal mismatch | Deal not matching position | Verify ownership |
| 319 | InvalidTriggerPrice | Invalid trigger price | Unrealistic trigger value | Use valid price |
| 320 | HasActivePositions | Has active positions | Open positions exist | Close positions first |
| 321 | TooManyDeals | Too many deals | Exceeds MAX_DEALS (100) | Close existing deals |
| 322 | InvalidFreeze | Invalid freeze | Frozen amount invalid | Ensure freeze ≤ size |
| 323 | InvalidPositionState | Invalid position state | State inconsistent | Contact support |

---

# Characteristics

- Enforced during deal creation, modification, and closure  
- Protect state transitions  
- Enforce deal limits and trigger integrity  
- Validate ownership and position consistency  
