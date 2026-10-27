# ADR-0011: Next.js BFF with httpOnly refresh cookie

* Status: Accepted · Date: 2026-10-28 · Deciders: Product owner, Tech lead

## Context
Avoid XSS token theft and keep the offline cache free of credentials.

## Decision
Next route handlers proxy auth; the refresh token lives in an httpOnly Secure SameSite cookie; the access token stays in memory; CSRF protections; API is called with a bearer token from memory.

## Consequences
+ Safer browser story. − Extra hop and cookie handling; documented.
