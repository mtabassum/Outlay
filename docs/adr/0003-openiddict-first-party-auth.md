# ADR-0003: OpenIddict with first-party login-ticket grant

* Status: Accepted · Date: 2026-10-28 · Deciders: Product owner, Tech lead

## Context
Need OAuth2/OIDC, JWT, refresh rotation, revocation, RBAC, an MFA step, workspace selection and a backend-only sessionMinutes option. Duende IdentityServer is commercial; OpenIddict is open source and embeddable.

## Decision
Embed OpenIddict. Login is a short facade (/api/auth/login → /mfa/verify → workspace) ending with a **one-time login ticket** exchanged at /connect/token via a custom grant. Refresh tokens rotate with reuse detection. The Next.js BFF keeps the refresh token in an httpOnly cookie. Upgrade path: Authorization Code + PKCE with a hosted login page.

## Consequences
+ Simple SPA integration, full control of MFA/session. + Standards-based tokens. − Not the OAuth 2.1 recommended browser flow (mitigated by BFF + short-lived single-use tickets); the custom grant needs careful tests.
