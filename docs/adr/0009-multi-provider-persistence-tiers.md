# ADR-0009: Multi-provider persistence with support tiers

| Status | Date | Deciders |
|---|---|---|
| Accepted | 2026-10-27 | Mehnaz Tabassum, Md Ikramul Islam Siddique Porosh |

## Context

The owner requires the DB to be switchable by configuration among InMemory, SQL Server, MySQL, Oracle, PostgreSQL, MS Access and MongoDB, with EF Core (Dapper optional). These providers differ greatly. As of October 2026: Pomelo MySQL has **no EF Core 10 build** (Oracle's `MySql.EntityFrameworkCore` 10.x and the `Microting` fork exist); the MongoDB EF Core provider (10.x) lacks Select projections, migrations and foreign keys; MS Access (Jet) needs the Windows-only ACE OLE DB driver.

## Decision

- `Database:Provider` selects an `IDatabaseProvider` strategy from per-provider assemblies (`Outlay.Persistence.<Provider>`), each declaring `DatabaseCapabilities`.
- **Tiers:** Tier 1 (SQL Server, PostgreSQL: full CI on every PR), Tier 2 (MySQL via Oracle provider, Oracle: nightly), Tier 3 (MongoDB, MS Access: smoke tests, documented limitations), plus InMemory for dev/demo.
- EF Core everywhere. Dapper only for relational report reads behind `IReportReadStore`. MongoDB reports via the driver's aggregation pipeline. `ProjectSafe` handles providers without server projection.
- Provider assemblies can be excluded with MSBuild switches, so a lagging provider never blocks an EF Core/.NET upgrade.

## Consequences

+ Meets the requirement honestly, with clear expectations per provider.
- Significant test matrix and migrations per relational provider.
- Some features degrade on Tier 3 (documented in TRD §6.2).

## Alternatives considered

Single provider (simplest, but violates the requirement); Dapper-only data layer (loses EF features and Mongo support).
