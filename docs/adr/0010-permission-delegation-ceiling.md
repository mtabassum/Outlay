# ADR-0010: Permission delegation with ceiling and cascade

* Status: Accepted · Date: 2026-10-28 · Deciders: Product owner, Tech lead

## Context
Super Admin → Admin → Manager/Member must delegate only what they hold; an Admin must not exceed the granted scope.

## Decision
Permissions are resource:action codes. Grants store grantedBy. A grant is valid only if ⊆ grantor's effective set and the grantor outranks the target. Revocation cascades. A PermissionVersion claim invalidates caches/tokens.

## Consequences
+ Least privilege, auditable. − Cascade logic needs thorough tests; the UI must explain cascades.
