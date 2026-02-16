---
id: errors-orderbook
title: OrderBook Errors (280–299)
sidebar_position: 13
---

# OrderBook Errors (280–299)

These errors apply to order book account management and internal order indexing logic.

---

# Error Table

| Code | Name | Message | Cause | Solution |
|------|------------------------------|----------------------------------|----------------------------------------------|----------------------------------------------|
| 280 | InvalidOrderBookOperation | Invalid order book operation | Operation invalid for current state | Verify order book state |
| 281 | OrderBookNotFound | Order book not found | Order book account missing | Initialize order book |
| 282 | OrderBookFull | Order book is full | At maximum capacity | Wait for orders to clear |
| 283 | OrderBookEmpty | Order book is empty | No orders present | Add orders first |
| 284 | DuplicateOrderId | Duplicate order ID | ID already exists | Use unique ID |
| 285 | OrderIdNotFound | Order ID not found | ID does not exist | Verify order ID |
| 286 | InvalidOrderBookState | Invalid order book state | State corrupted | Contact support |
| 287 | OrderBookIndexOutOfBounds | Order book index out of bounds | Index exceeds size | Use valid index |
| 288 | OrderBookCapacityExceeded | Order book capacity exceeded | Too many orders | Wait for space |

---

# Characteristics

- Triggered during order insertion, removal, or lookup  
- Enforce capacity limits  
- Validate order ID uniqueness  
- Protect order book indexing integrity  
