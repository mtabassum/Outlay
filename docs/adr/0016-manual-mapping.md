# ADR-0016: Manual mapping instead of AutoMapper

| Status | Date | Deciders |
|---|---|---|
| Accepted | 2026-10-27 | Mehnaz Tabassum, Md Ikramul Islam Siddique Porosh |

## Context

Every slice maps between domain entities and DTOs (responses, list rows, import rows, assistant data). AutoMapper is the traditional choice in ASP.NET Core codebases. The facts below, as of October 2026, shaped this decision.

1. **Licensing.** AutoMapper **15.0+** (July 2025) is dual-licensed RPL-1.5 / Lucky Penny Software commercial, the same model as MediatR (ADR-0003). **14.0.0** (Feb 2025) is the last MIT release. The Community licence is free only under US$5M revenue, for non-profits, education and non-production. Outlay is meant for real production use, so this is a second licence to track.
2. **Security.** CVE-2026-32933 / GHSA-rvv3-g6hj-g44x (CVSS 7.5, High) is a denial of service: deeply nested object graphs cause uncontrolled recursion, a `StackOverflowException` and **process termination**. It affects every version before **15.1.1** and 16.0.0–16.1.0. The fixes (15.1.1, 16.1.1) are in the **commercial** lines only. **14.0.0 is unpatched** (restore shows NU1903). So "stay on the free version" means shipping a known high-severity vulnerability.
3. **Runtime failure modes.** Convention mapping is resolved at runtime. A renamed property becomes a silently unmapped field unless every profile has an `AssertConfigurationIsValid()` test. "Find all references" and rename refactoring don't see the mapping.
4. **Over-exposure risk.** Convention mapping copies anything whose names match. When a new entity field is added, it can start appearing in API responses without anyone deciding it should. In a multi-tenant financial app, a response must contain exactly the fields chosen for it (no amounts to Super Admin, no internal flags).
5. **Inbound mapping is against our design.** Entities are created and changed through domain methods (`Expense.Record(...)`, `Approve(...)`) that enforce business rules (ADR-0002). Mapping DTO → entity with a mapper would bypass those methods. Even AutoMapper's author recommends using it only for outbound flattening.
6. **Projections and providers.** EF queries should project to DTOs in SQL. MongoDB's EF provider doesn't support `Select` projections, so Outlay uses `ProjectSafe(caps, expression)` (TRD §6.4). That works with a plain `Expression<Func<TEntity, TDto>>`, and a hand-written expression is visible, testable and provider-agnostic.
7. **Vertical slices keep DTOs small and local.** Most slices map 4–10 properties in one place. A central mapping profile would couple slices together (ADR-0002).

## Decision

- Map **explicitly**, next to the slice: `ToDto()` extension methods for in-memory mapping, and a static `Expression<Func<TEntity, TDto>> Projection` for query projection (`ProjectSafe` or `Select`).
- **Inbound** data never maps onto entities. Commands call domain factory and behaviour methods.
- No mapping library in Application or Domain.
- **Escape hatch:** if a feature needs a large amount of repetitive mapping (e.g. wide export rows), use **Mapperly** (`Riok.Mapperly`, Apache-2.0). It's a source generator: mapping code is generated at compile time, it's readable, it warns on unmapped members, and it uses no reflection at runtime. Using it in a feature requires a one-line note in that feature's folder, not a new ADR.

## Consequences

**Positive**
- No commercial licence and no unpatched CVE in the dependency tree.
- Mapping errors are compile errors. Rename and find-references cover all mappings.
- Every API response field is a deliberate choice, which supports tenant and role data minimisation.
- Projections are plain expressions that translate on every provider, including the MongoDB fallback path.
- Easy for new developers and AI agents: there's no mapping configuration to discover.

**Negative**
- More lines of code: roughly 5–15 per DTO.
- A forgotten new field won't appear in a DTO until someone adds it. That's acceptable, and intended (point 4).

**Neutral**
- Slice-level tests already assert DTO content, so no separate mapping tests are needed.

## Alternatives considered

| Option | Why not (now) |
|---|---|
| AutoMapper 15/16 (commercial or Community licence) | Licence tracking for production use, runtime mapping errors, over-exposure risk. Viable only if the owner's company qualifies for and accepts the Community licence. |
| AutoMapper 14.0.0 (last MIT) | Contains CVE-2026-32933 (High, DoS) with no free fix. Rejected. |
| Mapster (MIT) | Capable and fast, but still convention-based with runtime configuration. Not needed given small slice DTOs (YAGNI). |
| Mapperly (Apache-2.0, source generator) | Good fit if mapping volume grows. Kept as the approved escape hatch above. |

## References

- Jimmy Bogard, *AutoMapper and MediatR Commercial Editions Launch Today* (2 July 2025)
- NVD, CVE-2026-32933 (AutoMapper uncontrolled recursion)
- ADR-0002 (Clean Architecture + VSA), ADR-0003 (MediatR licensing), TRD §6.4 (`ProjectSafe`)
