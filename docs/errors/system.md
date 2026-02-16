---
id: errors-system
title: System Errors (240–279)
sidebar_position: 12
---

# System Errors (240–279)

These errors relate to global protocol state, feature flags, batch operations, and high-level system controls.

---

# Error Table

| Code | Name | Message | Cause | Solution |
|------|-------------------------------|-------------------------------|----------------------------------------------|----------------------------------------------|
| 240 | UnsupportedAsset | Unsupported asset | Asset type not supported | Use supported asset |
| 241 | SystemPaused | System paused | Protocol paused by admin | Wait for unpause |
| 242 | FeatureNotImplemented | Feature not implemented | Feature unavailable | Await future update |
| 243 | InvalidInstructionData | Invalid instruction data | Malformed instruction payload | Verify encoding |
| 244 | OperationNotStarted | Operation not started | Operation not initialized | Initialize first |
| 245 | OperationExpired | Operation expired | Deadline passed | Restart with new deadline |
| 246 | InvalidDuration | Invalid duration | Duration parameter invalid | Provide valid duration |
| 247 | InvalidBatchSize | Invalid batch size | Exceeds max batch size | Reduce batch size |
| 248 | InvalidPaginationParams | Invalid pagination params | Offset/limit invalid | Correct pagination values |
| 249 | ProgramPaused | Program paused | `Config.isPaused = true` | Wait for unpause |
| 250 | BatchProcessingFailed | Batch processing failed | Batch execution error | Check batch parameters |
| 251 | InsufficientClosedAmount | Insufficient closed amount | Not enough closed positions | Close more positions |
| 252 | OrderNotMatched | Order not matched | Could not match order | Adjust price or wait |
| 253 | EmergencyWithdrawalNotAllowed | Emergency withdrawal not allowed | Conditions not satisfied | Use normal withdrawal |
| 254 | NoRewards | No rewards available | No rewards to claim | Claim later |

---

# Characteristics

- Triggered by global protocol flags  
- Enforce pause and safety controls  
- Validate batch and time-based operations  
- Protect feature gating and system invariants  
