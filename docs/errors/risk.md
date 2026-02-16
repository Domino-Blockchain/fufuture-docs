---
id: errors-risk-management
title: Risk Management Errors (160–179)
sidebar_position: 8
---

# Risk Management Errors (160–179)

These errors enforce margin safety, leverage limits, and liquidation logic.

---

# Error Table

| Code | Name | Message | Cause | Solution |
|------|------------------------------|----------------------------------|----------------------------------------------|----------------------------------------------|
| 160 | PositionAtRisk | Position at risk | Position health below safe threshold | Add margin or reduce size |
| 161 | OrderNotAtRisk | Order not at risk | Does not meet liquidation criteria | Cannot liquidate healthy order |
| 162 | LiquidationThresholdExceeded | Liquidation threshold exceeded | Margin ratio below maintenance rate | Add margin urgently |
| 163 | InvalidMarginAmount | Invalid margin amount | Margin insufficient for size | Increase margin |
| 164 | RiskCheckFailed | Risk check failed | Operation would cause undercollateralization | Cannot withdraw/reduce margin |
| 165 | InvalidLiquidationThreshold | Invalid liquidation threshold | Config threshold invalid | Admin must correct setting |
| 166 | InvalidLeverage | Invalid leverage | Zero or above max leverage | Use valid leverage range |
| 167 | LeverageTooHigh | Leverage too high | Exceeds allowed maximum | Reduce leverage |
| 168 | LiquidationFailed | Liquidation failed | Liquidation process error | Retry liquidation |

---

# Characteristics

- Triggered during open, modify, withdraw, or liquidation flows  
- Enforce maintenance margin constraints  
- Prevent undercollateralized positions  
- Protect system solvency  
- Bound leverage to configured limits  
