# OAuth Flow Core

## Purpose

Defines the provider-agnostic core flow for OAuth 2.0 Authorization Code with PKCE and for OpenID Connect flows built on top of it.

## Relationship to Core Rules

Global security, execution order, stop conditions, and document precedence are defined in:

- `../../../core/SECURITY_INVARIANTS.md`
- `../../../core/EXECUTION_RULES.md`
- `../../../core/STOP_CONDITIONS.md`
- `../../../core/DECISION_MODEL.md`

This file defines only the reusable OAuth/OIDC flow pattern. Skill orchestration belongs in `../AGENT_EXECUTION_SPEC.md`.

## When to Use

Use this file for every social-auth implementation that relies on OAuth 2.0 or OIDC.

## Core Flow

1. Generate `state` for every login attempt.
2. Generate `nonce` when OIDC applies.
3. Generate `code_verifier` for PKCE.
4. Derive `code_challenge` from the verifier using `S256`.
5. Persist pre-auth state in a backend-controlled store with:
   - provider binding
   - short TTL
   - one-time-use semantics
6. Redirect to the provider authorization endpoint with:
   - `response_type=code`
   - `client_id`
   - `redirect_uri`
   - `scope`
   - `state`
   - `code_challenge`
   - `code_challenge_method=S256`
   - `nonce` when OIDC applies
7. On callback, validate:
   - provider match
   - `state`
   - TTL / expiry
   - one-time-use status
   - `nonce` later during ID token validation when OIDC applies
8. Exchange the authorization code using the original `code_verifier`.
9. Validate token response and ID token claims when OIDC applies.
10. Load or derive the provider profile.
11. Link or create the local user according to account-linking governance.
12. Rotate or create a server-side session.
13. Persist tokens server-side only when the architecture requires storage.
14. Invalidate the pre-auth state after successful or terminal failure handling.

## Required Validations

### Authorization Request
- use Authorization Code flow only
- use PKCE with `S256`
- bind state to the provider and login attempt
- request only necessary scopes

### Callback
- reject missing or mismatched `state`
- reject expired or reused pre-auth state
- reject callbacks for the wrong provider route
- fail safely when provider returns an explicit error

### Token Response
- validate the provider token response shape
- reject incomplete or malformed responses
- validate ID token signature and claims when OIDC applies

## OIDC-Specific Rules

When OIDC applies:

- `nonce` is required
- validate ID token signature
- validate `iss`
- validate `aud`
- validate `exp`
- validate `iat` where relevant
- validate `nonce`
- validate `azp` when relevant to the provider

See `jwks-validation-helper.md` for lightweight validation support where appropriate.

## Pre-Auth State Requirements

Pre-auth state should include only what is necessary to complete the flow safely, for example:

- provider
- state
- nonce when needed
- code verifier
- created-at / expiry timestamp
- one-time-use marker or equivalent replay protection

Store this in a backend-controlled session, encrypted cookie, or server store depending on the selected architecture.

## Error Handling Expectations

- do not create a session on validation failure
- do not retry the authorization code exchange blindly
- do not continue when provider identity data is incomplete unless governance explicitly allows it
- return a safe redirect or safe error response without leaking sensitive details

## Output Implications

A correct implementation based on this pattern should make it easy to identify:

- where pre-auth state is stored
- how callback validation works
- where token exchange occurs
- how identity linking is handled
- where session rotation occurs

## Maintenance Rule

- reusable OAuth/OIDC flow logic belongs here
- provider-specific variations belong in provider docs
- framework-specific route and session details belong in adapter docs
