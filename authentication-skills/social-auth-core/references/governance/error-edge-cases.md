# Error and Edge Case Governance

## Purpose

Defines how social-auth-specific errors and edge cases must be handled.

## Relationship to Core Rules

Global error handling and security rules are defined in:

- `../../../core/SECURITY_INVARIANTS.md`
- `../../../core/STOP_CONDITIONS.md`

This file defines only domain-specific edge-case handling for social authentication.

## Callback Error Cases

The implementation must account for:

- missing `code`
- missing `state`
- mismatched `state`
- missing `nonce` where OIDC applies
- mismatched `nonce`
- provider returns an explicit error
- expired or reused authorization code
- callback arriving on the wrong route or environment

## Provider Response Edge Cases

The implementation must handle:

- token endpoint failure
- incomplete token response
- missing user profile data
- missing email
- unverified email
- invalid or incomplete ID token claims
- provider-specific response shape differences

## Identity Edge Cases

The implementation must handle:

- duplicate provider identity
- conflicting account linkage
- existing user with matching email but different linked identity
- provider identity already linked elsewhere
- provider returns data insufficient for safe linking

## Session Edge Cases

The implementation must handle:

- session not rotated after successful login
- callback replay attempt
- stale or inconsistent session state
- callback received after login context has expired

## Required Behaviours

The system must:

- fail closed on validation errors
- avoid creating a session on failure
- avoid retrying authorization code exchange blindly
- avoid proceeding with incomplete identity data unless policy explicitly allows it
- return safe, non-sensitive user-facing error responses

## Abuse Handling

The implementation should treat the following as abuse or attack indicators when repeated:

- invalid or repeated `state` mismatches
- repeated invalid callbacks
- replay-like callback behaviour
- rapid provider failure loops on login endpoints

Rate limiting and safe monitoring should be applied where appropriate.

## User-Facing vs Internal Handling

- user-facing errors must remain generic and safe
- internal logs must not contain secrets, raw tokens, or full provider payloads
- internal audit logs should record the reason a flow failed without leaking sensitive material
- account-linking denials should be explained safely without revealing protected account data

## Maintenance Rule

- social-auth-specific edge-case governance belongs here
- generic error handling and trust-boundary rules belong in `core/`
- provider-specific response anomalies should be documented in provider docs if unique
