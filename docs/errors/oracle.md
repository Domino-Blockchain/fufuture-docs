---
id: errors-price-oracle
title: Price & Oracle Errors (100–119)
sidebar_position: 6
---

# Price & Oracle Errors (100–119)

These errors occur during price resolution, oracle validation, and slippage checks.

---

# Error Table

| Code | Name | Message | Cause | Solution |
|------|--------------------------|-------------------------------|--------------------------------------------------|------------------------------------------------|
| 100 | InvalidPrice | Invalid price | Price zero, negative, or unreasonable | Provide valid positive price |
| 101 | StalePriceData | Oracle price too stale | Timestamp exceeds staleness threshold | Wait for oracle update |
| 102 | InvalidOracleAccount | Invalid oracle account | Account not valid price feed | Verify correct Pyth/Switchboard feed |
| 103 | OraclePriceOutOfBounds | Oracle price out of bounds | Price unreasonably high/low | Check oracle status |
| 104 | SlippageExceeded | Slippage exceeded | Price moved beyond tolerance | Increase slippage or retry |
| 105 | InvalidClosePrice | Invalid close price | Close price invalid | Retry with current market price |
| 106 | InvalidPriceFeed | Invalid price feed | Feed account invalid/corrupted | Verify configured oracle address |
| 107 | StalePriceFeed | Stale price feed | Feed not recently updated | Wait for fresh update |
| 108 | LowConfidencePriceFeed | Low confidence price feed | Oracle confidence too wide | Wait for stable price data |
| 109 | NoPriceFeedConfigured | No price feed configured | Asset missing oracle config | Admin must configure feed |
| 110 | InvalidOracleConfig | Invalid oracle config | Configuration invalid | Update oracle settings |
| 111 | InvalidPriceStatus | Invalid price status | Oracle status unavailable | Wait for normal oracle status |

---

# Characteristics

- Triggered during order execution or liquidation
- Enforce oracle freshness and confidence checks
- Protect against price manipulation
- Enforce slippage tolerance constraints
