# ADR-0007: Delegated, subset-only permission model

| Status | Date | Deciders |
|---|---|---|
| Accepted | 2026-10-27 | Mehnaz Tabassum, Md Ikramul Islam Siddique Porosh |

## Context

Roles alone can't express the owner's rule: Super Admin grants Admins selected features; an Admin creates Managers and Users and can grant only what the Admin holds; each person sees only what they were granted. The requirement also defines data visibility (Members see own, Managers see team).

## Decision

- **Permissions** are the authorisation unit (catalogue in code, e.g. `expenses.approve`). **Role templates** (SuperAdmin, TenantAdmin, Manager, Member) give defaults.
- `permission_grants` rows (Grant/Deny) carry `granted_by_membership_id`.
- **Subset rule:** a grant succeeds only if the grantor's effective permissions contain every permission granted.
- **Hierarchy:** `memberships.parent_membership_id`. Targets must be below the actor.
- **Cascade revocation:** removing a permission removes it from everyone who received it through that person (recursive query), with a preview first.
- **Data scope** permissions: `expenses.read.own|team|all`, applied by `IDataScopeService`.
- Effective sets are cached per `permission_version`, which is bumped on every change and enforced via the `pver` token claim.

## Consequences

+ Precisely matches the brief, auditable, and a UI can be built on it (S-16).
+ Approvals respect data scope: the approver group for an expense is all Admins plus Managers with `expenses.approve` whose scope covers the member (`IApproverResolver`, added 2026-10-28).
- More complex than plain RBAC. Mitigated by a single service and exhaustive tests.

## Alternatives considered

Plain role-based checks (can't express delegation); policy-based ABAC engine (overkill, YAGNI).
