# ADR-0004: Minimal API endpoints per slice

| Status | Date | Deciders |
|---|---|---|
| Accepted | 2026-10-27 | Mehnaz Tabassum, Md Ikramul Islam Siddique Porosh |

## Context

Endpoints should be thin and live next to their slice concept. Controllers group many actions per class, which works against VSA and pulls in MVC overhead.

## Decision

Each use case gets one class implementing `IEndpoint { void Map(IEndpointRouteBuilder app); }`, discovered by assembly scanning and grouped with `MapGroup("/api/v1/<resource>")`. Each must declare name, summary, description, tags, typed `Produces*`, filters (idempotency) and authorization metadata. An architecture test fails the build when summary or description is missing.

## Consequences

+ One file per endpoint, easy to find. Full OpenAPI metadata. Good performance.
- Developers used to controllers need a short orientation (Backend Dev Guide).

## Alternatives considered

MVC controllers grouped per resource (familiar, but heavier and less aligned with slices); FastEndpoints (extra dependency).
