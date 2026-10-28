# ADR-0005: OpenIddict + ASP.NET Core Identity for OAuth 2.0 / OIDC

| Status | Date | Deciders |
|---|---|---|
| Accepted | 2026-10-27 | Mehnaz Tabassum, Md Ikramul Islam Siddique Porosh |

## Context

The owner requires OAuth 2.0, Microsoft Identity and OpenIddict, plus token rotation and revocation, MFA by OTP, lockout, and an optional API-only session duration. Duende IdentityServer needs a commercial licence for most companies. OpenIddict is Apache-2.0.

## Decision

- **ASP.NET Core Identity** manages users, lockout (3 attempts), security stamps and reset tokens, with a BCrypt hasher (ADR-0006).
- **OpenIddict 7** is the authorization server in the API process: password grant (first-party clients only) + custom `urn:outlay:otp` grant (MFA step 2) + `urn:outlay:switch_tenant` + rolling `refresh_token` + revocation + end-session + discovery. Authorization Code + PKCE is enabled for future third-party/SSO clients.
- Access tokens: 10 minutes, encrypted and signed. Refresh tokens are rotated on every use, and reuse outside a 30 s leeway revokes the whole authorization.
- The SPA's refresh token lives in an `HttpOnly; Secure; SameSite=Strict` cookie. The access token is kept in memory.
- `sstamp`/`pver` claims are validated against cached values, so revocation takes effect within 60 s.
- `session_duration` on the password grant sets an absolute session expiry (`sexp`), clamped by config. The UI never sends it.

## Consequences

+ Standards-based, free, extensible, and testable from Postman.
- The password grant is discouraged for third-party clients. This is mitigated by restricting it to first-party clients and keeping PKCE available.
- The auth server lives in the API process. It can be split out later without changing clients.

## Alternatives considered

Duende IdentityServer (licence cost); plain JWT issuing (no standard rotation/revocation/discovery); external IdP only (no offline/dev control, and the dev OTP requirement).
