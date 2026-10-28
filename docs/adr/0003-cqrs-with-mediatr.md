# ADR-0003: CQRS with MediatR

| Status | Date | Deciders |
|---|---|---|
| Accepted | 2026-10-27 | Mehnaz Tabassum, Md Ikramul Islam Siddique Porosh |

## Context

The owner requires CQRS with MediatR. Cross-cutting concerns (validation, authorization, tenancy, caching, transactions, logging) must apply consistently without bloating handlers. **Licensing:** MediatR 13.0+ (July 2025) is dual-licensed RPL-1.5 / commercial (Lucky Penny Software). A free Community licence covers organisations under US$5M gross revenue (and under US$10M outside capital), non-profits, education and non-production use. 12.5.0 is the last Apache-2.0 release. A missing key only logs a warning.

## Decision

- Use **MediatR 14.x** with a Community licence key, configured via secrets as `MediatR:LicenseKey`. Confirmed 2026-10-28: the owner's company qualifies (Q-06). The Community licence covers the **current** release with all fixes; it isn't the frozen 12.5.0. Re-check eligibility if revenue passes US$5M or outside capital passes US$10M.
- Commands and queries implement Outlay's own markers (`ICommand<T>`, `IQuery<T>`, `IRequirePermission`, `ICacheableQuery`, `IIdempotentCommand`, `ITransactionalCommand`).
- Pipeline order: Logging → Validation → Authorization → Tenant/PlanLimit → Caching (queries) → Transaction (commands).

## Consequences

+ Uniform cross-cutting behaviour. Thin, testable handlers.
+ Because slices depend on our markers and `ISender`, replacing MediatR (e.g. with the MIT source-generated `Mediator` library) is mechanical if the licence situation changes.
- Indirection: navigating from endpoint to handler needs IDE "go to implementation".
- A licence obligation must be tracked.

## Alternatives considered

Plain service classes (simpler, but cross-cutting concerns repeat); pinning MediatR 12.5.0 (free, but no fixes); source-generated Mediator (MIT, good fallback).
