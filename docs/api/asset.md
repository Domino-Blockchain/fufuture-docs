---
id: asset
title: Asset (Underlying) API Reference
sidebar_position: 6
---

# Asset (Underlying) API Reference

This section documents how to initialize and manage an underlying asset.

Underlying assets define:

- Trading configuration
- Fee parameters
- Oracle configuration
- Collateral eligibility
- Funding fee logic
- Margin logic

---

# PDA

```
["asset", name]
```

Example:

```
["asset", "BTC"]
```

Derivation:

```ts
function findAssetAddress(programId: PublicKey, name: string): [PublicKey, number] {
  return PublicKey.findProgramAddressSync(
    [Buffer.from('asset'), Buffer.from(name, 'utf-8')],
    programId
  );
}
```

---

# UnderlyingAsset State Layout

## Rust Struct

```
pub struct UnderlyingAsset {
    pub tag: [u8; 8],
    pub is_initialized: bool,
    pub is_enabled: bool,
    pub is_collateral: bool,
    pub name: String,
    pub asset_type: AssetType,
    pub trading_fee_rate: u64,
    pub liquidation_fee_rate: u64,
    pub formula_program: BorshPubkey,
    pub price_feed_config: PriceFeedConfig,
    pub price_feed_id: [u8; 32],
    pub price_decimals: u8,
    pub min_price: u128,
    pub max_price: u128,
    pub price_third_flag: bool,
    pub third_proxy_price: u128,
    pub authority: BorshPubkey,
    pub created_at: i64,
    pub updated_at: i64,
    pub bump: u8,
    pub min_order_size: u128,
    pub reward_gas: Option<u64>,
}
```

---

# Instruction: SetUnderlying (Initialize Asset)

## Discriminator

```
30
```

## Purpose

Creates and initializes an `UnderlyingAsset` account.

Also updates:

- ConfigAssets PDA
- ConfigPairs PDA

---

## Required Accounts

| Index | Account | Signer | Writable |
|--------|---------|---------|----------|
| 0 | authority | ✅ | ✅ |
| 1 | asset PDA | ❌ | ✅ |
| 2 | formula program | ❌ | ❌ |
| 3 | System Program | ❌ | ❌ |
| 4 | Rent Sysvar | ❌ | ❌ |
| 5 | ConfigAssets PDA | ❌ | ✅ |
| 6 | ConfigPairs PDA | ❌ | ✅ |

---

## Instruction Data Layout

```
u8    discriminator (30)
u32   name length
bytes name
u8    leverage
u128  initial_price (E18)
u8    enabled flag
```

---

## TypeScript Instruction Builder

```ts
function createSetUnderlyingInstruction(
  programId: PublicKey,
  authority: PublicKey,
  assetPda: PublicKey,
  formulaProgram: PublicKey,
  name: string,
  leverage: number,
  initPrice: string,
  enabled: boolean
): TransactionInstruction {

  const nameBytes = Buffer.from(name, 'utf-8');
  const nameLengthBuffer = Buffer.alloc(4);
  nameLengthBuffer.writeUInt32LE(nameBytes.length, 0);

  const data = Buffer.concat([
    Buffer.from([30]),
    nameLengthBuffer,
    nameBytes,
    Buffer.from([leverage]),
    writeU128LE(initPrice),
    Buffer.from([enabled ? 1 : 0]),
  ]);

  const [configAssetsPDA] = findConfigAssetsAddress(programId);
  const [configPairsPDA] = findConfigPairsAddress(programId);

  return new TransactionInstruction({
    keys: [
      { pubkey: authority, isSigner: true, isWritable: true },
      { pubkey: assetPda, isSigner: false, isWritable: true },
      { pubkey: formulaProgram, isSigner: false, isWritable: false },
      { pubkey: SystemProgram.programId, isSigner: false, isWritable: false },
      { pubkey: SYSVAR_RENT_PUBKEY, isSigner: false, isWritable: false },
      { pubkey: configAssetsPDA, isSigner: false, isWritable: true },
      { pubkey: configPairsPDA, isSigner: false, isWritable: true },
    ],
    programId,
    data,
  });
}
```

---

# Example: Initialize BTC Asset

```ts
const ASSET_NAME = 'BTC';
const ASSET_LEVERAGE = 10;
const BTC_INITIAL_PRICE = '50000000000000000000000'; // $50,000 in E18

const [assetPda] = findAssetAddress(PROGRAM_ID, ASSET_NAME);

const setUnderlyingIx = createSetUnderlyingInstruction(
  PROGRAM_ID,
  authority.publicKey,
  assetPda,
  Keypair.generate().publicKey,
  ASSET_NAME,
  ASSET_LEVERAGE,
  BTC_INITIAL_PRICE,
  true
);

const tx = new Transaction().add(setUnderlyingIx);

await sendAndConfirmTransaction(
  connection,
  tx,
  [authority]
);
```

---

# Asset Behavior & Logic

## Validation Rules

Asset validates:
```
- tag must equal `ASSETCFG`
- is_initialized must be true
- name length within bounds
- fee rates must be > 0 and <= 1e18
- price feed config must pass validation
```
---

## Price Logic

`get_price()` resolves in this order:

1. If `price_third_flag == true`
   → use `third_proxy_price`
2. Else
   → use `price_feed_config`
3. Else
   → error

---

## Funding Fee

```
position_value = amount * price / 1e18
funding_fee = position_value * base_rate / 1e18
```

Base rate default: `0.01% per day`

---

## Fee Calculation

Fees include:

- Trading fee
- Funding fee
- Liquidation fee

All scaled in E18.

---

## Margin Logic

```
Public Pool:  10%
Private Pool: 15%
Force fee:    5%
```

Decimals are normalized to token decimals.

---

# Asset Management Methods

The asset supports runtime updates:

- set_enabled()
- set_collateral()
- set_asset_type()
- set_formula()
- set_trading_fee_rate()
- set_liquidation_fee_rate()
- set_price_bounds()
- set_price_third_proxy()
- update_price_feed_config()

All require asset authority.

---

# Asset Types

```
Stablecoin
Native
Volatile (default)
```

---

# PriceFeedConfig Structure

```
pub struct PriceFeedConfig {
    pub feed_type: u8,
    pub feed_address: Pubkey,
    pub decimals: u8,
    pub max_staleness: u64,
    pub min_confidence: u128,
    pub price: u128,
    pub last_updated: i64,
}
```

Feed Types:

```
0 = None
1 = Chainlink
2 = Pyth
3 = Manual
```

---

# Asset Account Size

```
UnderlyingAsset::LEN
```

Includes:

- discriminator
- full config
- price config
- optional reward gas
- padding

---

# Summary

Asset initialization:

1. Derive PDA
2. Send SetUnderlying
3. Account created
4. Added to ConfigAssets
5. Pair registered in ConfigPairs

Asset controls:

- Fees
- Oracle config
- Margin logic
- Trading enablement
- Collateral enablement
- Funding calculation
