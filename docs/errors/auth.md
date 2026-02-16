---
id: errors-authorization
title: Authorization Errors (20–39)
sidebar_position: 3
---

# Authorization Errors (20–39)

These errors occur when a signer, authority, or keeper does not meet required permission constraints.

---

# Error Table

| Code | Name | Message | Cause | Solution |
|------|----------------------|---------------------------|----------------------------------------------|----------------------------------------------|
| 20 | Unauthorized | Unauthorized | User lacks permission for operation | Check signer is correct account owner |
| 21 | NotAuthorizedKeeper | Not authorized keeper | Signer not in authorized keepers list | Add keeper to `Config.authorizedKeepers` |
| 22 | UnauthorizedKeeper | Unauthorized keeper | Keeper not authorized for operation | Use authorized keeper account |
| 23 | InvalidAuthority | Invalid authority | Authority does not match expected | Verify authority equals `Config.owner` |
| 24 | MissingSignature | Missing signature | Required signer did not sign | Ensure required accounts sign transaction |
| 25 | UnauthorizedSigner | Unauthorized signer | Signer not permitted for operation | Verify correct account is signing |
| 26 | InvalidTokenOwner | Invalid token owner | Token account owner mismatch | Confirm token account ownership |
| 27 | UnauthorizedPoolAccess | Unauthorized pool access | User not permitted to access pool | Validate pool access permissions |

---

# Characteristics

- Triggered by signer validation failures
- Enforced via authority checks and access control logic
- Not recoverable without correct signer/permissions
- Usually indicates incorrect account passed or wrong wallet signing
