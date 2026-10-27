# ADR-0005: Multi-tenancy: shared database with TenantId discriminator

* Status: Accepted · Date: 2026-10-28 · Deciders: Product owner, Tech lead

## Context
Many small tenants, strict isolation, simple ops, must work on every provider including Access and Mongo.

## Decision
Single schema; every tenant table has TenantId. Enforce via EF global filters, a Dapper TenantSql helper that requires @TenantId, Mongo repository filters, tenant-prefixed cache keys, and a mandatory isolation test suite. Tenant id comes only from the token.

## Consequences
+ Simple, cheap, portable. − A bug could leak across tenants → mitigated by tests, analyzers, optional Postgres RLS. Per-tenant DB remains a future option for Enterprise.
