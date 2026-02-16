---
id: error-usage
title: Error Usage Examples
sidebar_position: 17
---

# Usage Examples

## Basic Error Handling

```ts
import { Connection, Transaction, Keypair } from '@solana/web3.js';
import { extractProgramError } from './errors';

async function executeTradeWithErrorHandling(
    connection: Connection,
    transaction: Transaction,
    signers: Keypair[]
) {
    try {
        const signature = await connection.sendTransaction(transaction, signers);
        await connection.confirmTransaction(signature);

        console.log('Trade executed successfully:', signature);
        return signature;

    } catch (error) {
        const perpetualError = extractProgramError(error);

        if (perpetualError) {
            console.error('Trading Error:', {
                code: perpetualError.code,
                name: perpetualError.name,
                message: perpetualError.message,
                category: perpetualError.category
            });

            switch (perpetualError.code) {
                case 2:   // InsufficientFunds
                    throw new Error('Please deposit more collateral before trading');

                case 120: // InsufficientLiquidity
                    throw new Error('Pool lacks liquidity - try smaller trade size');

                case 162: // LiquidationThresholdExceeded
                    throw new Error('Position would be liquidated - add margin first');

                case 361: // TradingPaused
                    throw new Error('Trading is currently paused by admin');

                default:
                    throw new Error(`Trading failed: ${perpetualError.message}`);
            }
        }

        throw error;
    }
}
````

---

## Error Recovery Strategies

```ts
/**
 * Retry with exponential backoff for transient errors
 */
async function executeWithRetry(
    fn: () => Promise<any>,
    maxRetries: number = 3
): Promise<any> {
    for (let i = 0; i < maxRetries; i++) {
        try {
            return await fn();
        } catch (error) {
            const perpetualError = extractProgramError(error);

            const nonRetryableErrors = [
                2,   // InsufficientFunds
                40,  // OrderNotFound
                162, // LiquidationThresholdExceeded
                361, // TradingPaused
            ];

            if (perpetualError && nonRetryableErrors.includes(perpetualError.code)) {
                throw error;
            }

            if (i < maxRetries - 1) {
                const delay = Math.pow(2, i) * 1000;
                await new Promise(resolve => setTimeout(resolve, delay));
                continue;
            }

            throw error;
        }
    }

    throw new Error('Retry attempts exhausted');
}
```

---

## User-Friendly Error Messages

```ts
/**
 * Convert technical error to user-friendly message
 */
function getUserFriendlyErrorMessage(error: any): string {
    const perpetualError = extractProgramError(error);

    if (!perpetualError) {
        return 'An unexpected error occurred. Please try again.';
    }

    const userMessages: Record<number, string> = {
        2:   'You don\'t have enough funds. Please deposit more collateral.',
        20:  'You don\'t have permission to perform this action.',
        21:  'This operation requires an authorized keeper.',
        40:  'The order you\'re looking for doesn\'t exist.',
        45:  'Your order size is too small. Minimum order size is required.',
        46:  'Your order size is too large. Please reduce it.',
        80:  'This limit order has expired. Please create a new one.',
        100: 'Invalid price provided.',
        101: 'Price data is too old. Please wait for oracle update.',
        120: 'There isn\'t enough liquidity in the pool for this trade.',
        162: 'This action would put your position at risk of liquidation.',
        167: 'The leverage you requested is too high.',
        207: 'Your deposit is too small. Please deposit more.',
        313: 'No position found. Please open a position first.',
        361: 'Trading is currently paused. Please try again later.',
    };

    return userMessages[perpetualError.code] || perpetualError.message;
}
```

---

## Logging and Monitoring

```ts
/**
 * Log error for monitoring
 */
function logError(error: any, context: string) {
    const perpetualError = extractProgramError(error);

    if (perpetualError) {
        console.error('Program Error:', {
            context,
            code: perpetualError.code,
            name: perpetualError.name,
            message: perpetualError.message,
            category: perpetualError.category,
            timestamp: new Date().toISOString()
        });

        // Example: send to monitoring service
        // analytics.track('program_error', { ... });
    } else {
        console.error('Unknown Error:', {
            context,
            error: error.toString(),
            timestamp: new Date().toISOString()
        });
    }
}
```