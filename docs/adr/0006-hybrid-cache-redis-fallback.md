# ADR-0006: HybridCache with Redis L2 and graceful fallback

* Status: Accepted · Date: 2026-10-28 · Deciders: Product owner, Tech lead

## Context
Need speed, no DB hits when cached, a Redis backplane, but the app must keep working if Redis is off.

## Decision
Use HybridCache (L1 memory + L2 Redis) behind ICacheService; Redis pub/sub as backplane for L1 invalidation; circuit breaker + fallback to L1/in-memory; optional Memcached adapter. Tag-based invalidation from domain events.

## Consequences
+ Stampede protection, resilient. − Eventual-consistency window (seconds), documented. Cache is never the source of truth.
