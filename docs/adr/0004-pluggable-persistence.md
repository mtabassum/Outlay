# ADR-0004: Pluggable persistence: provider × ORM

* Status: Accepted · Date: 2026-10-28 · Deciders: Product owner, Tech lead

## Context
Must switch among InMemory, SQL Server, MySQL, Oracle, PostgreSQL, MS Access, MongoDB and between EF Core and Dapper via configuration.

## Decision
Application depends on repository/reader interfaces. Infrastructure has one implementation set per (provider, ORM) chosen at startup. A shared **contract test suite** runs against every provider. Dapper SQL is per-dialect via ISqlDialect. Access = Dapper/Windows only; Mongo = driver/EF Mongo. Default InMemory.

## Consequences
+ True portability, testable. − Highest maintenance cost in the project; Access and Oracle have limits (A-06); provider-specific features avoided. New repository methods require contract tests.
