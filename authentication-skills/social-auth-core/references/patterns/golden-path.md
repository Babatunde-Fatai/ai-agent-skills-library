# Golden Path

## Purpose

Provides one canonical end-to-end example path for implementing social authentication with minimal ambiguity.

## Relationship to Core Rules

This file is illustrative. It does not replace:

- `../AGENT_EXECUTION_SPEC.md`
- `oauth-flow-core.md`
- provider docs
- adapter docs

## When to Use

Use this file when you want a single compact example of how the main pieces fit together after discovery is complete.

## Example Scope

This example assumes:

- one provider
- one backend-controlled callback route
- server-side pre-auth state
- server-side session creation
- safe redirect back into the application

Adapt the same structure to the target framework and provider.

## Canonical Flow

1. Confirm provider, framework, callback ownership, and session model.
2. Create login start route.
3. Generate `state`, `nonce` when needed, and `code_verifier`.
4. Persist pre-auth state server-side.
5. Redirect to the provider authorization endpoint.
6. Handle callback on a backend-owned route.
7. Validate `state` and pre-auth state.
8. Exchange authorization code for tokens using the original verifier.
9. Validate ID token when OIDC applies.
10. Load provider profile data if required.
11. Link or create the local user.
12. Rotate or create the application session.
13. Redirect to a safe post-login destination.
14. Invalidate pre-auth state.

## Minimal File Plan Example

A typical implementation may create or update:

- start route
- callback route
- provider helper
- pre-auth state helper
- account-linking service
- session creation / rotation service
- auth-state hydration endpoint if the frontend requires it

## What This Pattern Intentionally Omits

This pattern does not hardcode:

- a specific framework
- a specific ORM
- a specific provider endpoint set
- a specific session library

Those choices must come from discovery, provider docs, and adapter docs.

## Maintenance Rule

- this file should remain illustrative and compact
- detailed rules belong in the core layer, execution spec, provider docs, and adapter docs
