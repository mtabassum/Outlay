# ADR-0018: React + Vite + Tailwind v4 + TanStack frontend

| Status | Date | Deciders |
|---|---|---|
| Accepted | 2026-10-27 | Mehnaz Tabassum, Md Ikramul Islam Siddique Porosh |

## Context

The frontend must use the latest React, stay upgradable, use Tailwind, be offline-first, a PWA, bilingual and accessible, with a unique professional design.

## Decision

React 19 + TypeScript strict + Vite; Tailwind CSS v4 with `@theme` tokens ("Kantha Ledger"); Radix primitives; React Router 7; TanStack Query (server state) and Table; Zustand (client state); React Hook Form + Zod; Dexie; i18next + ICU; vite-plugin-pwa (`injectManifest` → `service-worker.js`, own `manifest.json`); Recharts; Vitest/RTL/MSW and Playwright + axe. API types are generated from OpenAPI. Renovate handles upgrades.

## Consequences

+ Mainstream, well-maintained libraries with clear upgrade paths.
+ Design tokens isolate visual changes.
- Many small libraries. Mitigated by Renovate grouping and a documented structure.

## Alternatives considered

Next.js (SSR not needed for an authenticated PWA); Redux Toolkit (more boilerplate than needed).
