# Token Management

## Purpose

Defines reusable rules for storing, refreshing, revoking, and auditing provider tokens in social-auth integrations.

## Relationship to Core Rules

Global token-safety expectations are defined in:

- `../../../core/SECURITY_INVARIANTS.md`

This file defines only reusable token-management patterns.

## When to Use

Use this file when the implementation stores access tokens, refresh tokens, or token metadata server-side.

## Storage Rules

- store tokens server-side only
- encrypt sensitive token values at rest when feasible
- keep token metadata separate and queryable where useful
- avoid treating provider tokens as frontend session state

Useful metadata may include:

- provider
- granted scopes
- issued-at
- expires-at
- refresh-token expiry where relevant
- token type
- subject / provider account reference

## Access Token Handling

- do not persist access tokens longer than the architecture requires
- prefer refreshing or re-fetching rather than treating expired tokens as valid
- avoid sending raw provider access tokens to the browser

## Refresh Token Handling

- request refresh-capable scopes only when necessary
- use rotation where the provider supports it
- treat refresh token reuse as a high-signal security event
- see `refresh-token-lifecycle.md` for lifecycle-specific rules

## Revocation and Logout

On logout or disconnect where relevant:

- revoke provider tokens if supported and required
- delete or invalidate local token records appropriately
- clear the local session independently of provider revocation success

## Scope Discipline

- record the actual granted scopes
- compare granted scopes with expected scopes when this affects application behaviour
- avoid silently assuming broader access than was granted

## Failure Handling

- token refresh failure should usually lead to re-auth or re-link flow, not blind retries
- revocation failure should not be treated as proof of active authentication
- expired or missing token metadata should not be ignored

## Maintenance Rule

- reusable token rules belong here
- provider-specific token quirks belong in provider docs
- framework-specific storage details belong in adapter docs where relevant
