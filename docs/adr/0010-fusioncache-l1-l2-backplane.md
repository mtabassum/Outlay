# ADR-0010: FusionCache L1/L2 with backplane

| Status | Date | Deciders |
|---|---|---|
| Accepted | 2026-10-27 | Mehnaz Tabassum, Md Ikramul Islam Siddique Porosh |

## Context

Reads should hit cache before the DB. Redis must be optional (fallback to memory, or Memcached). With several API replicas, in-process caches must be invalidated across nodes. A Redis outage must not take the API down.

## Decision

Use **FusionCache** (MIT) behind Outlay's `IAppCache`: L1 memory always; L2 = Redis, Memcached (any `IDistributedCache`) or none (`Cache:L2`); Redis **backplane** for cross-node L1 invalidation; **fail-safe** (serve stale on failure); distributed-cache circuit breaker; stampede protection; tagging. Keys and tags are always tenant-prefixed (`CacheKeys`). Caching is applied by `CachingBehavior` / `CacheInvalidationBehavior` after commit.

## Consequences

+ Resilient by design. Configurable. Graceful Redis-down behaviour.
- One more library to learn (wrapped by `IAppCache`).

## Alternatives considered

Microsoft HybridCache (good, but no built-in backplane for cross-node L1 invalidation); raw IDistributedCache (no L1, no fail-safe).
