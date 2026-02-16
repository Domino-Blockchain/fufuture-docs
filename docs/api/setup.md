---
id: setup
title: Setup & Configuration
sidebar_position: 1
---

# Setup & Configuration

This section provides the required libraries and basic connection configuration for interacting with the on-chain **Fufuture** program.

For full instruction routing, refer to:

- `src/processor/mod.rs` (main instruction router)
- Rust test files for correct account ordering examples
- Futures Error Reference for detailed error codes

---

## Required Libraries

```bash
npm install @solana/web3.js @solana/spl-token bn.js @coral-xyz/borsh
````

---

## Basic Connection Setup

```ts
import { Connection, PublicKey, Transaction, Keypair } from '@solana/web3.js';
import * as borsh from '@coral-xyz/borsh';

// 1. Connect to RPC endpoint
const connection = new Connection(
  'https://api.devnet.domichain.io',
  'confirmed'
);

// 2. Program ID (deployed contract address)
const PROGRAM_ID = new PublicKey('HJPxiV5zcxRnUrYMBfGVCnQhmKQ35v5YBxdJ4NHuab5z');

// 3. User wallet
const userWallet = Keypair.fromSecretKey(
  Uint8Array.from([
    /* your secret key */
  ])
);
```
