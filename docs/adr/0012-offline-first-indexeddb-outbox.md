# ADR-0012: Offline-first with IndexedDB outbox and idempotency keys

| Status | Date | Deciders |
|---|---|---|
| Accepted | 2026-10-27 | Mehnaz Tabassum, Md Ikramul Islam Siddique Porosh |

## Context

The frontend must keep data and CRUD operations in IndexedDB when the backend is down and sync everything when it's back, without duplicates.

## Decision

- **Dexie** database per user+tenant. Network-first reads with IndexedDB fallback.
- Writes while offline (or failing) go to an **outbox** with a client-generated UUIDv7 **Idempotency-Key** and the base `rowVersion`. Optimistic local records use temporary ids.
- Replay is FIFO on reconnect/focus/interval/Background Sync. Server responses map to synced / conflict (409) / rejected (422/403) / retry (5xx).
- **Delta pull** by per-tenant `change_version` (`GET /sync/changes`).
- Server-wins by default, with a conflict UI. Business rules (budgets, hard limits) are always evaluated on the server at sync time.
- No tokens in IndexedDB. Data is cleared on logout unless the user opts to keep it.

## Consequences

+ Works on flaky networks. No duplicate transactions.
- Users may see "pending" items later rejected. Mitigated by clear status and notifications.

## Alternatives considered

Online-only with retry (fails the requirement); full CRDT/replication (overkill).
