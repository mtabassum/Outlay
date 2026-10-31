# ADR-0013: Deterministic (rule-based) voice/text assistant with guardrails

| Status | Date | Deciders |
|---|---|---|
| Accepted | 2026-10-27 | Mehnaz Tabassum, Md Ikramul Islam Siddique Porosh |

## Context

The owner wants a voice and text assistant that answers questions about expenses, is **deterministic**, and never discloses another tenant's or Super Admin's data. Admins may ask about the Managers and Users under them. US-22 adds that the chatbot must follow the same access rules as the screens.

## Decision

- Server-side intent engine: normalise (EN/BN, Bengali digits, month names) → rule-based intent match → slot resolution (period via `IClock` in the tenant time zone; people resolved **only within the asker's data scope**) → `AssistantGuard` → execute the **existing authorised MediatR queries** → localised answer templates.
- No generative model, so there's no free text and no prompt injection surface. Identical input over identical data gives an identical answer.
- Client: Web Speech API (`SpeechRecognition` en-US/bn-BD, `speechSynthesis`), feature-detected, with text always available.
- Rate limited and audited (intent only, no amounts).

## Consequences

+ Safe by construction: guardrails reuse the platform's authorisation.
+ Predictable and testable.
- Phrasing coverage is limited to the catalogue. Mitigated with suggestions/help and by growing the catalogue.
- Browser speech support varies (no Firefox recognition).

## Alternatives considered

LLM with RAG (non-deterministic, harder guardrails, data egress); LLM intent classifier only (possible future amendment, ROADMAP v2.0).
