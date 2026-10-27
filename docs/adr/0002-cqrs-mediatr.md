# ADR-0002: CQRS with MediatR behind our own markers

* Status: Accepted · Date: 2026-10-28 · Deciders: Product owner, Tech lead

## Context
Requirement: CQRS via MediatR. Cross-cutting concerns (logging, validation, authorization, idempotency, caching, transactions) should not be repeated in handlers. Newer MediatR major versions carry commercial licence terms (A-08).

## Decision
Use MediatR pipeline behaviors. Handlers depend on ICommand<T>/IQuery<T> markers that wrap MediatR types. **Pin a licensed/compatible version (12.x is Apache-2.0); verify terms before upgrading.**

## Consequences
+ Uniform pipeline, small handlers. + Swapping the mediator (e.g., a source-generated Mediator) is a mechanical change. − Indirection; mitigated by naming conventions and tests.
