---
id: errors-liquidity-pools
title: Liquidity & Pool Errors (120–159)
sidebar_position: 7
---

# Liquidity & Pool Errors (120–159)

These errors occur during pool initialization, liquidity provision, withdrawals, and order matching.

---

# Error Table

| Code | Name | Message | Cause | Solution |
|------|--------------------------|-------------------------------|------------------------------------------------|------------------------------------------------|
| 120 | InsufficientLiquidity | Insufficient liquidity | Pool lacks liquidity to match trade | Reduce trade size or wait |
| 121 | PoolNotFound | Pool not found | Pool account missing | Initialize pool first |
| 122 | InvalidPoolState | Invalid pool state | Pool in incorrect state | Ensure pool is active |
| 123 | PoolMismatch | Pool mismatch | Wrong pool for pair | Verify pool PDA |
| 124 | NoLiquidityPools | No liquidity pools available | No pools configured | Admin must create pools |
| 125 | MinimumLiquidityNotMet | Minimum liquidity not met | Below required threshold | Wait for LP deposits |
| 126 | PoolSizeExceeded | Pool size exceeded | Pool at max capacity | Wait for capacity |
| 127 | AmountTooSmall | Amount less than minimum | Below pool minimum | Increase amount |
| 128 | PoolAlreadyExists | Pool already exists | PDA already initialized | Use existing pool |
| 129 | InvalidPoolType | Invalid pool type | Not public/private | Specify valid type |
| 130 | InvalidPoolConfiguration | Invalid pool configuration | Invalid init parameters | Check config values |
| 131 | PoolRejectingOrders | Pool rejecting orders | `isRejectOrder=true` | Pool not accepting orders |
| 132 | InsufficientLPTokens | Insufficient LP tokens | Not enough LP shares | Reduce withdrawal |
| 133 | MakerNotFound | Maker not found | Maker not registered | Add liquidity first |
| 134 | MakerAlreadyExists | Maker already exists | Maker already registered | Maker already active |
| 135 | UnauthorizedMaker | Unauthorized maker | Not allowed in private pool | Use authorized maker |
| 136 | PoolRefused | Pool refused | Pool refused match | Adjust order or pool |
| 137 | InvalidDecimals | Invalid decimals | Token decimals mismatch | Verify token decimals |
| 138 | InvalidPoolAddress | Invalid pool address | PDA mismatch | Verify derivation |
| 139 | EscrowAccountNotAllowed | Escrow account not allowed | Operation not permitted | Use correct account |
| 150 | InvalidPoolMode | Invalid pool mode | Mode invalid | Use public/private |
| 151 | InsufficientShares | Insufficient shares | LP shares too low | Withdraw less |
| 152 | RemainingAmountTooSmall | Remaining amount too small | Residual balance too small | Withdraw all or less |
| 153 | NoMakerAvailable | No maker available | No maker to match | Wait for liquidity |
| 154 | TooManyOrders | Too many orders | Exceeds max order count | Close existing orders |
| 155 | EscrowAccountNotFound | Escrow account not found | Escrow missing | Initialize escrow first |

---

# Characteristics

- Enforced during liquidity add/remove flows  
- Enforced during pool matching  
- Protect pool capacity and share accounting  
- Validate correct pool PDAs and modes  
- Enforce maker authorization in private pools  

