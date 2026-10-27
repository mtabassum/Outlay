# ADR-0009: Deterministic rule-based assistant

* Status: Accepted · Date: 2026-10-28 · Deciders: Product owner, Tech lead

## Context
The assistant must answer expense questions by voice/text, never leak other tenants' or higher-privilege data, and be predictable.

## Decision
Intent lexicon + parser (EN/BN) mapped to existing MediatR queries under the caller's permissions. No LLM in v1; IAssistantEngine allows a future adapter (off by default). Voice via the browser Web Speech API.

## Consequences
+ Safe, testable, cheap. − Limited phrasing coverage; mitigated by lexicon growth and helpful fallbacks.
