---
id: pda
title: Program Derived Addresses (PDAs)
sidebar_position: 2
---

# Program Derived Addresses (PDAs)

Program Derived Addresses (PDAs) are deterministic accounts generated from a set of seeds and a program ID. They do not have private keys. Instead, the on-chain program signs for them using `invoke_signed`.

PDAs are used to:

- Store program-owned state
- Guarantee deterministic account locations
- Prevent collisions via unique seed combinations
- Enforce program-level authority

All PDAs in Fufuture are derived using `PublicKey.findProgramAddressSync()` with fixed seed prefixes.

---

## PDA Derivation Helper Functions

```ts
import { PublicKey } from '@solana/web3.js';
```

---

### Config PDAs

#### ConfigCore PDA

```ts
export function findConfigCoreAddress(programId: PublicKey): [PublicKey, number] {
  return PublicKey.findProgramAddressSync(
    [Buffer.from('config')],
    programId
  );
}
```

#### ConfigAssets PDA

```ts
export function findConfigAssetsAddress(programId: PublicKey): [PublicKey, number] {
  return PublicKey.findProgramAddressSync(
    [Buffer.from('config'), Buffer.from('assets')],
    programId
  );
}
```

#### ConfigPairs PDA

```ts
export function findConfigPairsAddress(programId: PublicKey): [PublicKey, number] {
  return PublicKey.findProgramAddressSync(
    [Buffer.from('config'), Buffer.from('pairs')],
    programId
  );
}
```

#### ConfigKeepers PDA

```ts
export function findConfigKeepersAddress(programId: PublicKey): [PublicKey, number] {
  return PublicKey.findProgramAddressSync(
    [Buffer.from('config'), Buffer.from('keepers')],
    programId
  );
}
```

---

### User State PDAs

#### User Account PDA

```ts
export function findUserAccountAddress(
  programId: PublicKey,
  user: PublicKey,
  tokenMint: PublicKey
): [PublicKey, number] {
  return PublicKey.findProgramAddressSync(
    [
      Buffer.from('user_account'),
      user.toBuffer(),
      tokenMint.toBuffer()
    ],
    programId
  );
}
```

---

### Trading PDAs

#### Deal PDA

```ts
export function findDealAddress(
  programId: PublicKey,
  dealId: bigint
): [PublicKey, number] {
  const buffer = Buffer.alloc(8);
  buffer.writeBigUInt64LE(dealId);

  return PublicKey.findProgramAddressSync(
    [Buffer.from('deal'), buffer],
    programId
  );
}
```

#### Position PDA

```ts
export function findPositionAddress(
  programId: PublicKey,
  user: PublicKey,
  pair: string,
  direction: number
): [PublicKey, number] {
  return PublicKey.findProgramAddressSync(
    [
      Buffer.from('position'),
      user.toBuffer(),
      Buffer.from(pair),
      Buffer.from([direction])
    ],
    programId
  );
}
```

#### Limit Order PDA

```ts
export function findLimitOrderAddress(
  programId: PublicKey,
  orderId: bigint
): [PublicKey, number] {
  const buffer = Buffer.alloc(8);
  buffer.writeBigUInt64LE(orderId);

  return PublicKey.findProgramAddressSync(
    [Buffer.from('limit_order'), buffer],
    programId
  );
}
```

---

### Pool & Asset PDAs

#### Pool PDA

```ts
export function findPoolAddress(
  programId: PublicKey,
  tokenMint: PublicKey,
  underlyingName: string
): [PublicKey, number] {
  return PublicKey.findProgramAddressSync(
    [
      Buffer.from('pool'),
      tokenMint.toBuffer(),
      Buffer.from(underlyingName)
    ],
    programId
  );
}
```

#### Asset PDA

```ts
export function findAssetAddress(
  programId: PublicKey,
  name: string
): [PublicKey, number] {
  return PublicKey.findProgramAddressSync(
    [Buffer.from('asset'), Buffer.from(name)],
    programId
  );
}
```

---

## PDA Design Notes

### 1. Seed Structure

Each PDA begins with a fixed namespace prefix:

* `"config"`
* `"user_account"`
* `"deal"`
* `"position"`
* `"limit_order"`
* `"pool"`
* `"asset"`

This prevents cross-structure collisions.

---

### 2. Determinism

Given the same:

* Program ID
* Seeds
* Seed order

The derived PDA will always be identical.

Changing seed order changes the PDA.

---

### 3. BigInt Encoding

All numeric IDs (`dealId`, `orderId`) are encoded as:

* 8 bytes
* Little-endian
* `writeBigUInt64LE`

This must match the Rust side exactly.

---

### 4. String Seeds

Strings such as:

* `pair`
* `underlyingName`
* `asset name`

are converted using:

```
Buffer.from(string)
```

These must match byte-for-byte with Rust’s `as_bytes()` usage.

---

### 5. Bump Seed

`findProgramAddressSync` returns:

```
[publicKey, bump]
```

The bump is used internally when signing via `invoke_signed` on-chain.

The frontend typically does not need to store the bump unless constructing CPI logic.

---

### 6. Authority Model

PDAs have:

* No private key
* No externally owned authority
* Program-only signing capability

This guarantees state integrity and prevents external control.

---

All state accounts in Fufuture are PDA-based and must be derived exactly as shown above to avoid instruction failures.
