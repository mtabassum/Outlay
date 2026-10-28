# ADR-0008: Shared-database multi-tenancy with TenantId filters

| Status | Date | Deciders |
|---|---|---|
| Accepted | 2026-10-27 | Mehnaz Tabassum, Md Ikramul Islam Siddique Porosh |

## Context

Many small organisations share the platform, and the data must be completely private. Users may belong to several organisations.

## Decision

One database, `tenant_id` on every tenant-owned table (first column of every index). Tenant comes from the token claim only. EF Core 10 **named query filters** (`Tenant`, `SoftDelete`) apply automatically, and a SaveChanges interceptor stamps and guards `tenant_id`. Cache keys, SignalR groups, file paths, search, import, export and the assistant are tenant-scoped. Super Admins live in a Platform pseudo-tenant. An automated isolation suite exercises every endpoint with two tenants.

## Consequences

+ Cheapest to operate. Simple migrations. Works across all providers.
- A missed filter could leak data. Mitigated by defence in depth (filter + interceptor + architecture test + isolation suite).
- Noisy-neighbour risk. Mitigated by per-tenant rate limiting.

## Alternatives considered

Database-per-tenant (strong isolation, high ops cost, and hard with 7 providers); schema-per-tenant (provider-specific).
