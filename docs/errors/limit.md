---
id: errors-limit-orders
title: Limit Order Errors (80–99)
sidebar_position: 5
---

# Limit Order Errors (80–99)

These errors apply specifically to limit order lifecycle and trigger validation.

---

# Error Table

| Code | Name | Message | Cause | Solution |
|------|--------------------------|----------------------------|----------------------------------------------|------------------------------------------------|
| 80 | LimitOrderExpired | Limit order expired | Current time exceeds `goodTill` timestamp | Create new order with later expiry |
| 81 | LimitOrderPriceMismatch | Limit order price mismatch | Order price does not match trigger condition | Verify trigger price |
| 82 | PriceConditionNotMet | Price condition not met | Market price not yet reached trigger level | Wait for trigger price |
| 83 | InvalidLimitOrder | Invalid limit order | Parameters invalid | Validate price, amount, direction, expiry |
| 84 | LimitOrderNotActive | Limit order not active | State not `PENDING` or `PARTIAL` | Cannot operate on completed/cancelled order |
| 85 | LimitOrderAlreadyExists | Limit order already exists | Order ID already in use | Use next available order ID |

---

# Characteristics

- Triggered during limit order creation or execution
- Validate expiry timestamps
- Enforce trigger price conditions
- Ensure correct limit order state transitions
