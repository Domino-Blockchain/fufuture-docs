---
id: errors-fee-funding
title: Fee & Funding Errors (180–199)
sidebar_position: 9
---

# Fee & Funding Errors (180–199)

These errors relate to funding calculations, fee configuration, and migration logic.

---

# Error Table

| Code | Name | Message | Cause | Solution |
|------|------------------------------|-------------------------------|----------------------------------------------------------|------------------------------------------------|
| 180 | InvalidFundingFee | Invalid funding fee | Funding fee calculation invalid | Verify funding parameters |
| 181 | InvalidMigrationTime | Invalid migration time | Timestamp invalid or in past | Use valid future timestamp |
| 182 | InvalidMigrationPeriod | Invalid migration period | Not in valid set (1–24 divisor of 24) | Use 1, 2, 3, 4, 6, 8, 12, or 24 |
| 183 | InvalidFeeRate | Invalid fee rate | Fee rate out of range | Provide reasonable fee rate |
| 184 | InvalidFundingRateMatrix | Invalid funding rate matrix | Matrix length not 365 | Provide full 365-day matrix |
| 185 | InsufficientFee | Insufficient fee | Fee amount too low | Increase fee amount |
| 186 | MigrationTooEarly | Migration too early | Period not elapsed | Wait for migration period |

---

# Characteristics

- Triggered during funding settlement  
- Validate migration scheduling  
- Enforce fee rate bounds  
- Ensure correct funding matrix configuration  
