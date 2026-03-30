# Testing and Validation Governance

## Purpose

Defines required testing and validation expectations for social authentication.

## Relationship to Core Rules

Global execution and security expectations are defined in the core layer and execution spec.

This file defines social-auth-specific validation requirements.

## Minimum Required Test Coverage

At minimum, the implementation should validate:

- `state` mismatch rejection
- `nonce` mismatch rejection where OIDC applies
- token exchange failure handling
- session creation only on success
- session rotation after successful login
- email verification policy enforcement

## Provider-Agnostic Tests

The implementation should cover:

- callback handling
- invalid request rejection
- replay attempt prevention
- invalid redirect handling
- login failure without session creation

## Account Linking Tests

The implementation should cover:

- verified email auto-link behaviour
- unverified email rejection for auto-link
- duplicate identity prevention
- conflict resolution handling
- already-linked provider identity rejection

## Session and Redirect Tests

The implementation should validate:

- session established only on successful verified login
- correct redirect after login success
- correct redirect or error handling after failure
- session not persisted after validation failure

## Mocking Strategy

Do not rely on real providers in CI/CD for baseline validation.

Use mocks or fakes for:

- discovery endpoints
- token exchange
- user info/profile fetches
- JWKS retrieval and validation where applicable

See:
`references/governance/minimal-test-harness.md`

## Release Readiness Criteria

The system is not ready unless:

- required auth-flow tests pass
- no session is created on failure paths
- linking policy is enforced correctly
- callback validation is strict
- provider-specific edge cases relevant to the chosen provider have been covered

## Maintenance Rule

- social-auth-specific testing rules belong here
- generic testing practice does not belong here unless it affects auth correctness directly
- provider-specific test exceptions should be documented in provider docs when needed
