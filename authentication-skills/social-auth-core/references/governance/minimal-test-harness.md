# Minimal Test Harness

## Purpose

Provides a minimal testing harness pattern for validating social authentication flows safely and consistently without depending on live third-party providers for baseline automated coverage.

## Relationship to Core Rules

This file provides testing utilities specific to social authentication.

Global testing expectations are defined in execution rules and the social-auth execution spec.

## Scope

This harness is intended to support:

- callback validation testing
- state mismatch testing
- nonce mismatch testing where OIDC applies
- token exchange failure handling
- safe account-linking branch coverage
- session creation and rotation validation

It is not intended to replace full staging or provider-console validation where those are required.

## Recommended Components

A minimal social-auth test harness should include:

- mocked provider authorization callback inputs
- mocked token endpoint responses
- mocked user info or profile responses
- mocked JWKS responses where ID token validation is required
- session inspection utilities
- redirect assertion helpers

## Minimum Scenarios to Simulate

The harness should make it easy to simulate:

- successful callback with valid state
- failed callback with missing state
- failed callback with mismatched state
- failed callback with missing or invalid nonce where applicable
- token endpoint timeout or provider error
- provider returns missing email
- duplicate identity / linking collision
- successful session creation and rotation
- failure path with no session creation

## Test Isolation Rules

- tests must not use real production secrets
- tests must not depend on live provider availability
- tests must isolate session state between cases
- tests must avoid persisting unsafe auth artifacts across test runs

## Output Expectations

The test harness should support assertions for:

- whether a session was created
- whether a session was rotated
- whether linking was allowed or rejected
- which redirect or response path was chosen
- whether validation failure prevented auth completion

## Maintenance Rule

- this file provides reusable auth-specific testing support
- it should stay lightweight and implementation-agnostic
- framework-specific test helpers should live closer to adapters if they become runtime-specific
