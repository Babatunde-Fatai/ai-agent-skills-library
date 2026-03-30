# Session Handling

## Purpose

Defines reusable session and pre-auth state handling patterns for social authentication.

## Relationship to Core Rules

Global security and cookie safety rules are defined in:

- `../../../core/SECURITY_INVARIANTS.md`

This file defines only the reusable session pattern for social-auth flows.

## When to Use

Use this file when:

- pre-auth state must survive a provider redirect
- a login callback must bind to a prior login attempt
- a session is created or rotated after successful authentication
- cookies are involved in the auth flow

## Pre-Auth State Pattern

A social-auth flow needs temporary pre-auth state before the real application session is established.

This pre-auth state commonly contains:

- provider
- `state`
- `nonce` when OIDC applies
- `code_verifier`
- issued-at / expiry metadata
- one-time-use or replay-protection marker

## Storage Options

### Server-Side Session Store
Use when:
- the framework already has session support
- you want stronger control over invalidation and TTL

### Encrypted / Signed Backend-Controlled Cookie
Use when:
- a server-side session store is unavailable or intentionally avoided
- the cookie can be strongly protected
- the TTL is short
- one-time-use semantics can still be enforced

### Dedicated Pre-Auth Store
Use when:
- auth flows are shared across services
- login state must be centrally managed
- you need explicit provider-bound transaction tracking

## Cookie Defaults for Session Identifiers

Where session identifiers are stored in cookies, prefer:

- `HttpOnly=true`
- `Secure=true`
- `SameSite=Lax` by default
- short scope and path where possible

Use `SameSite=None` only when the deployment topology truly requires it and the CSRF implications are understood.

See `samesite-decision-tree.md`.

## Session Rotation

After successful login:

- rotate the existing session identifier or issue a new session
- do not keep the pre-login session identifier as the authenticated session
- clear or invalidate pre-auth state after use

## Validation Requirements

- callback must not succeed without valid pre-auth state
- expired pre-auth state must be rejected
- reused pre-auth state must be rejected
- post-login session must be distinct from any unauthenticated pre-auth context

## Common Mistakes to Avoid

- storing provider tokens in browser-readable storage
- keeping long-lived pre-auth cookies
- using the main app session before auth completion is verified
- failing to rotate the session after login

## Maintenance Rule

- reusable session and pre-auth handling rules belong here
- framework-specific cookie or middleware details belong in adapter docs
