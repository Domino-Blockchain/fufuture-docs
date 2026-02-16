---
id: errors-general
title: General Errors (0–19)
sidebar_position: 2
---

# General Errors (0–19)

This category contains foundational protocol errors that are not tied to a specific subsystem.

These errors typically relate to:

- Instruction decoding
- Account validation
- Arithmetic safety
- PDA verification
- Serialization/deserialization

These should be treated as **critical validation failures**.

---

# Error Reference Table

| Code | Name | Message | Cause | Solution |
|------|--------|----------|--------|-----------|
| 0 | InvalidInstruction | Invalid instruction | Instruction discriminator not recognized | Check instruction data encoding |
| 1 | InvalidAccount | Invalid account | Account does not match expected type or state | Verify account PDA derivation and initialization |
| 2 | InsufficientFunds | Insufficient funds | User account lacks sufficient balance | Deposit more collateral before trading |
| 3 | MathOverflow | Arithmetic overflow | Calculation result exceeds maximum value | Reduce trade size or check input values |
| 4 | MathUnderflow | Arithmetic underflow | Calculation result below minimum value | Check calculation inputs for negative results |
| 5 | DivisionByZero | Division by zero | Attempted division by zero | Report to developers — indicates logic bug |
| 6 | InvalidPDA | Invalid PDA | PDA does not match expected derivation | Verify PDA seeds and program ID |
| 7 | AccountNotOwnedByProgram | Account not owned by program | Account owner does not match program ID | Ensure account was created by correct program |
| 8 | InvalidAccountData | Invalid account data | Account data cannot be deserialized | Account may be corrupted or wrong type |
| 9 | AccountAlreadyInitialized | Account already initialized | Attempted double initialization | Use existing account or new PDA seeds |
| 10 | InsufficientBalance | Insufficient balance | Available balance too low for operation | Close positions or deposit more funds |
| 13 | InvalidArgument | Invalid argument | Instruction parameter invalid | Validate parameter values and types |
| 14 | InstructionPackError | Error packing instruction data | Failed to serialize instruction | Check Borsh encoding |
| 15 | InstructionUnpackError | Error unpacking instruction data | Failed to deserialize instruction | Verify instruction format |

---

# Category Characteristics

General errors typically originate from:

- `require!()` validations
- Borsh unpack failures
- PDA seed mismatches
- Safe math wrappers
- Account owner checks
- Initialization guards

These errors should **never be ignored**.