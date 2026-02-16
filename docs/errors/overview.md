---
id: error-overview
title: Error Overview
sidebar_position: 1
---

# Error Reference

This section provides a comprehensive reference for all protocol-level errors emitted by the on-chain program, including:

- Error categorization
- Code ranges
- Frontend extraction strategies
- TypeScript mapping structures

All error definitions and numeric mappings are implemented in:

```

error.rs

````

---

# Error Handling Architecture

The protocol implements **365+ custom error codes**, organized into functional categories.

Errors are grouped into numeric ranges for rapid classification and debugging.

---

# Error Code Ranges

| Range | Category | Description |
|--------|------------|--------------|
| 0–19 | General Errors | Invalid instruction, insufficient funds, math errors |
| 20–39 | Authorization | Permission and signature validation errors |
| 40–79 | Order Management | Order creation, validation, and state issues |
| 80–99 | Limit Orders | Limit-order specific logic failures |
| 100–119 | Price & Oracle | Oracle resolution and price validation errors |
| 120–159 | Liquidity & Pools | Liquidity pool and deposit/withdraw errors |
| 160–179 | Risk Management | Margin, leverage, liquidation failures |
| 180–199 | Fees & Funding | Trading fees and funding rate errors |
| 200–219 | Configuration | Invalid system configuration parameters |
| 220–239 | Token Operations | SPL token transfer and account errors |
| 240–279 | System | System state and operational failures |
| 280–299 | Orderbook | Orderbook management errors |
| 300–349 | Deal & State | Deal lifecycle and state transition errors |
| 350–379 | Trading & Validation | Trade execution and validation errors |

---

# Error Data Structure

Frontend applications should map raw program errors into a structured interface.

## TypeScript Interface

```ts
interface PerpetualError {
    code: number;
    name: string;
    message: string;
    category: string;
}
````

---

## Error Categories Enum

```ts
enum ErrorCategory {
    GENERAL = "General",
    AUTHORIZATION = "Authorization",
    ORDER = "Order Management",
    LIMIT_ORDER = "Limit Orders",
    PRICE_ORACLE = "Price & Oracle",
    LIQUIDITY = "Liquidity & Pools",
    RISK = "Risk Management",
    FEES = "Fees & Funding",
    CONFIG = "Configuration",
    TOKEN = "Token Operations",
    SYSTEM = "System",
    ORDERBOOK = "Order Book",
    DEAL = "Deal & State",
    TRADING = "Trading & Validation"
}
```

---

# Extracting Errors From Transactions

When a transaction fails, the client must extract and normalize the error.

The protocol emits errors as:

* Numeric error codes
* Hex-encoded custom program errors
* Explicit `"Error Code: X"` patterns

---

## Extract Error Utility

```ts
/**
 * Extract error from failed transaction
 */
function extractProgramError(error: any): PerpetualError | null {
    // Direct numeric error
    if (error?.code && typeof error.code === 'number') {
        return getPerpetualError(error.code);
    }

    const message = error?.message || error?.toString() || '';

    // Match: custom program error: 0x...
    const customMatch = message.match(/custom program error: 0x([0-9a-fA-F]+)/);
    if (customMatch) {
        const errorCode = parseInt(customMatch[1], 16);
        return getPerpetualError(errorCode);
    }

    // Match: Error Code: X
    const codeMatch = message.match(/Error Code: (\d+)/);
    if (codeMatch) {
        const errorCode = parseInt(codeMatch[1], 10);
        return getPerpetualError(errorCode);
    }

    return null;
}
```

---

# Mapping Error Codes

Frontend applications should maintain a mapping table:

```ts
/**
 * Get error details from error code
 */
function getPerpetualError(code: number): PerpetualError {
    const errorInfo = ERROR_CODES[code];

    if (!errorInfo) {
        return {
            code,
            name: 'UnknownError',
            message: `Unknown error code: ${code}`,
            category: 'Unknown'
        };
    }

    return {
        code,
        name: errorInfo.name,
        message: errorInfo.message,
        category: errorInfo.category
    };
}
```

---

# Recommended Frontend Strategy

1. Catch transaction failure
2. Call `extractProgramError()`
3. Map error code using `ERROR_CODES`
4. Display normalized message to user
5. Log raw error for developer diagnostics

---

# Design Philosophy

The error system is:

* Deterministic
* Machine-parseable
* Categorized by subsystem
* Frontend-friendly
* Scalable to 300+ errors

This enables:

* Precise UX messaging
* Risk-aware UI flows
* Automated retry logic
* Better monitoring and analytics

---