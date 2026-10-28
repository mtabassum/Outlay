# ADR-0015: Validate-all-then-commit bulk import

| Status | Date | Deciders |
|---|---|---|
| Accepted | 2026-10-27 | Mehnaz Tabassum, Md Ikramul Islam Siddique Porosh |

## Context

Bulk CSV/Excel import must report **every** validation error and duplicate (by row and column) and roll back completely on any error. Some providers (InMemory) don't support transactions.

## Decision

Pipeline: read → parse → validate **all** rows (parallel, reusing the single-create FluentValidation validators) → detect duplicates in-file and against the DB (batched) → if any error, write nothing and return all errors plus an annotated workbook (red cells with comments) → otherwise commit in one transaction (chunked). Large files run as background jobs with SignalR progress.

## Consequences

+ Provider-independent "rollback" (nothing is written unless everything is valid).
+ One set of rules for single create, import and offline sync (DRY).
- Memory use for very large files. Mitigated by row/size limits and streaming readers.

## Alternatives considered

Row-by-row insert with transaction rollback (reports only the first error, and depends on provider transactions); partial imports (violate the requirement).
