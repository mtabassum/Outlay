# ADR-0001: Record architecture decisions

| Status | Date | Deciders |
|---|---|---|
| Accepted | 2026-10-27 | Mehnaz Tabassum, Md Ikramul Islam Siddique Porosh |

## Context

Outlay has many cross-cutting technical choices (architecture, auth, persistence, caching, offline). Both the owner and AI pair programmers need to know *why* a choice was made, so that it isn't silently undone.

## Decision

Use Architecture Decision Records (Michael Nygard format) in `docs/adr/NNNN-title.md`. Each ADR has Status (Proposed / Accepted / Superseded by NNNN / Deprecated), Context, Decision, Consequences and Alternatives considered. ADRs are immutable once accepted. To change a decision, write a new ADR that supersedes the old one, and update TRD §19.

## Consequences

+ Decisions are discoverable and reviewable in PRs.
+ AI agents are told (CLAUDE.md) to consult ADRs before changing architecture.
- A small overhead per decision.

## Alternatives considered

Wiki pages (drift from code); comments in code (not discoverable).
