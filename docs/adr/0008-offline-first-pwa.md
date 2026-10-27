# ADR-0008: Offline-first PWA with IndexedDB outbox

* Status: Accepted · Date: 2026-10-28 · Deciders: Product owner, Tech lead

## Context
Users must keep working with the backend down and sync later; install as an app; avoid duplicates on retry.

## Decision
Hand-written service-worker.js (shell + offline page), Dexie IndexedDB cache, an **outbox** of mutations with client GUID + Idempotency-Key, FIFO replay with backoff, conflict UI via RowVersion. Online-only actions: approvals, plan and permission changes. Tokens are never stored in IndexedDB.

## Consequences
+ Robust and understandable. − Conflict handling and evaluate-on-sync add UX complexity (AppFlow §11).
