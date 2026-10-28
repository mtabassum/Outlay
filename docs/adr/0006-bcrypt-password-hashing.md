# ADR-0006: BCrypt password hashing

| Status | Date | Deciders |
|---|---|---|
| Accepted | 2026-10-27 | Mehnaz Tabassum, Md Ikramul Islam Siddique Porosh |

## Context

The owner requires passwords to be "hashed and encrypted by bcrypt". ASP.NET Core Identity defaults to PBKDF2.

## Decision

Replace `IPasswordHasher<AppUser>` with `BcryptPasswordHasher` using **BCrypt.Net-Next** `EnhancedHashPassword` (SHA-384 pre-hash, which removes BCrypt's 72-byte input limit), with a configurable work factor (default 12) and `SuccessRehashNeeded` when the factor increases. Passwords are **hashed, never encrypted**: encryption is reversible and would weaken security. Data-at-rest encryption is provided at the database/disk level.

## Consequences

+ Meets the requirement with a well-reviewed adaptive hash.
+ Transparent upgrades of the work factor on login.
- About 250 ms per hash at factor 12 on reference hardware. The login rate limit protects against CPU exhaustion.

## Alternatives considered

Identity PBKDF2 default; Argon2id (strong, but not what the owner asked for).
