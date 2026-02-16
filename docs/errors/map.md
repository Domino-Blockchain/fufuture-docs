---
id: error-map
title: Complete Error Map
sidebar_position: 16
---

# Complete Error Map

This mapping combines all error ranges into a single lookup object.

It assigns:

- `name`
- `message`
- `category`
- optional `cause`
- optional `solution`

Frontend applications should use this unified map for deterministic error resolution.

---

## TypeScript Mapping

```ts
/**
 * Complete error code to error info mapping
 */
const ERROR_CODES: Record<number, {
    name: string;
    message: string;
    category: ErrorCategory;
    cause?: string;
    solution?: string;
}> = {
    // Combine all error ranges

    ...Object.entries(GENERAL_ERRORS).reduce((acc, [code, info]) => ({
        ...acc,
        [code]: { ...info, category: ErrorCategory.GENERAL }
    }), {}),

    ...Object.entries(AUTHORIZATION_ERRORS).reduce((acc, [code, info]) => ({
        ...acc,
        [code]: { ...info, category: ErrorCategory.AUTHORIZATION }
    }), {}),

    ...Object.entries(ORDER_ERRORS).reduce((acc, [code, info]) => ({
        ...acc,
        [code]: { ...info, category: ErrorCategory.ORDER }
    }), {}),

    ...Object.entries(LIMIT_ORDER_ERRORS).reduce((acc, [code, info]) => ({
        ...acc,
        [code]: { ...info, category: ErrorCategory.LIMIT_ORDER }
    }), {}),

    ...Object.entries(PRICE_ORACLE_ERRORS).reduce((acc, [code, info]) => ({
        ...acc,
        [code]: { ...info, category: ErrorCategory.PRICE_ORACLE }
    }), {}),

    ...Object.entries(LIQUIDITY_ERRORS).reduce((acc, [code, info]) => ({
        ...acc,
        [code]: { ...info, category: ErrorCategory.LIQUIDITY }
    }), {}),

    ...Object.entries(RISK_ERRORS).reduce((acc, [code, info]) => ({
        ...acc,
        [code]: { ...info, category: ErrorCategory.RISK }
    }), {}),

    ...Object.entries(CONFIG_ERRORS).reduce((acc, [code, info]) => ({
        ...acc,
        [code]: { ...info, category: ErrorCategory.CONFIG }
    }), {}),

    ...Object.entries(DEAL_ERRORS).reduce((acc, [code, info]) => ({
        ...acc,
        [code]: { ...info, category: ErrorCategory.DEAL }
    }), {}),

    ...Object.entries(TRADING_ERRORS).reduce((acc, [code, info]) => ({
        ...acc,
        [code]: { ...info, category: ErrorCategory.TRADING }
    }), {}),
};

export { ERROR_CODES, getPerpetualError, extractProgramError };
````

---

This file should be included in the frontend error handling layer and used by:

* `extractProgramError`
* `getPerpetualError`
* UI error normalization logic
