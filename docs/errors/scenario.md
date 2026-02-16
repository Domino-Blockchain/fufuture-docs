---
id: error-scenarios
title: Common Error Scenarios
sidebar_position: 18
---

# Common Error Scenarios

---

## Scenario 1: Opening Position Fails

```ts
// Error: InsufficientFunds (2)
// Cause: User balance too low
// Solution: Deposit collateral first

await deposit(connection, programId, user, tokenMint, amount);
````

---

## Scenario 2: Limit Order Not Executing

```ts
// Error: PriceConditionNotMet (82)
// Cause: Market price hasn't reached trigger
// Solution: Wait or recreate order at market price
```

---

## Scenario 3: Withdrawal Rejected

```ts
// Error: RiskCheckFailed (164)
// Cause: Withdrawal would cause liquidation
// Solution: Close positions or reduce withdrawal

const userInfo = await getUserAccountInfo(
    connection,
    programId,
    user,
    tokenMint
);

const maxWithdraw = userInfo.freeBalance;
```

---

## Scenario 4: Cannot Close Position

```ts
// Error: InsufficientAmount (304)
// Cause: Closing more than position size
// Solution: Close only available amount

const position = await getPositionsByPair(
    connection,
    programId,
    user,
    "BTC-USD"
);

const availableSize = position.long.size - position.long.freeze;
```

---

# Error Prevention Checklist

## Before Any Operation

* Config initialized (`InvalidAccount`)
* System not paused (`TradingPaused`)
* User account initialized (`InvalidUserAccount`)

---

## Before Trading

* Asset initialized & enabled (`AssetNotInitialized`, `AssetNotEnabled`)
* Pool exists & has liquidity (`PoolNotFound`, `InsufficientLiquidity`)
* Order size within limits (`OrderTooSmall`, `OrderTooLarge`)
* Sufficient balance (`InsufficientFunds`)
* Keeper authorized (`UnauthorizedKeeper`)
* Oracle price fresh (`StalePriceData`)

---

## Before Closing

* Position exists (`PositionNotFound`)
* Close amount ≤ available size (`InsufficientAmount`)
* Deal IDs valid (`DealNotFound`)

---

## Before Deposit / Withdrawal

* Amount ≥ minimum (`DepositAmountTooSmall`)
* Withdrawal won’t cause liquidation (`RiskCheckFailed`)

---

# Error Code Lookup

### Convert Hex Error to Decimal

```bash
echo $((16#7B))  # Output: 123
```

---

### Find Error by Code in Source

```bash
grep "= 123" error.rs
```

---