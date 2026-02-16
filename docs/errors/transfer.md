---
id: errors-token
title: Transfer & Token Errors (220–239)
sidebar_position: 11
---

# Transfer & Token Errors (220–239)

These errors occur during SPL token validation and transfer operations.

---

# Error Table

| Code | Name | Message | Cause | Solution |
|------|-------------------------------|------------------------------|----------------------------------------------|----------------------------------------------|
| 220 | TransferFailed | Transfer failed | Token transfer operation failed | Check balances and approvals |
| 221 | InvalidTokenAccount | Invalid token account | Account invalid or uninitialized | Verify token account exists |
| 222 | InvalidTokenMint | Invalid token mint | Mint does not match expected | Use correct token mint |
| 223 | TokenAccountNotOwnedByUser | Token account not owned by user | Owner mismatch | Use token account owned by signer |
| 224 | TokenTransferFailed | Token transfer failed | SPL transfer instruction failed | Verify balance and approvals |

---

# Characteristics

- Triggered during deposit, withdrawal, margin, and settlement  
- Enforce mint correctness  
- Enforce token account ownership  
- Depend on SPL token program execution  
