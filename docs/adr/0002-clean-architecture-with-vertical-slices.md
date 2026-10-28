# ADR-0002: Clean Architecture with Vertical Slices

| Status | Date | Deciders |
|---|---|---|
| Accepted | 2026-10-27 | Mehnaz Tabassum, Md Ikramul Islam Siddique Porosh |

## Context

The owner requires Clean Architecture *and* Vertical Slice Architecture, with code that stays simple and maintainable. Pure layered Clean Architecture scatters a feature over many folders. Pure VSA can let infrastructure leak into features.

## Decision

Combine both:
- **Projects** follow Clean Architecture: `Domain` (no dependencies) ← `Application` ← `Infrastructure`/`Persistence.*`; `Api` is the composition root.
- **Inside `Application`**, code is organised by **feature → use case** (`Features/Expenses/CreateExpense/`), each folder holding the command/query, handler, validator and DTOs.
- **Inside `Api`**, endpoint files mirror the same feature/use-case folders.
- Rules are enforced by NetArchTest (`Outlay.ArchitectureTests`).

## Consequences

+ A feature is understood by reading one folder. Adding a feature rarely touches shared code.
+ Domain rules stay testable and framework-free.
- Two folders per use case (Application + Api). Accepted for a clean dependency direction.

## Alternatives considered

Single-project VSA (faster to start, weaker boundaries); classic n-layer (feature scatter).
