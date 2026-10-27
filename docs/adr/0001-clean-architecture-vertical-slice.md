# ADR-0001: Clean Architecture with Vertical Slices

* Status: Accepted · Date: 2026-10-28 · Deciders: Product owner, Tech lead

## Context
We need strict dependency direction for portability (many DBs, ORMs, caches) yet fast feature delivery and easy onboarding. Pure layer-by-layer folders scatter one feature across many folders.

## Decision
Four projects (Domain, Application, Infrastructure, Api) obey the dependency rule; inside Application, code is organised by **feature slice** (command/query, handler, validator, DTO together). Cross-slice reuse only via Application abstractions. NetArchTest enforces the rules.

## Consequences
+ Changing infrastructure never touches business code. + A feature is findable in one folder. − Some duplication of small DTOs across slices (accepted; DRY applies to logic, not shape). − Needs discipline; architecture tests mitigate.
