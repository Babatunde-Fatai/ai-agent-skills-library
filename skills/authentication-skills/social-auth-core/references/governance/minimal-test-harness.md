# Minimal Test Harness Stub
**When to read:** When writing OAuth/OIDC tests without hitting real provider endpoints.
**What problem it solves:** Gives a minimal, repeatable way to mock discovery, JWKS, and token exchange.
**When to skip:** If you already have a working provider mock or full integration tests.
**Prerequisites:** Read `references/governance/testing-validation.md`.

## Minimal Stub Outline
1. Mock discovery URL to return `authorization_endpoint`, `token_endpoint`, and `jwks_uri`.
2. Host a JWKS endpoint that returns a test public key.
3. Mock token endpoint to assert `code`, `client_id`, `redirect_uri`, and `code_verifier`.
4. Return `access_token` and a signed `id_token` (RS256) using the test private key.

## Suggested Test Assertions
- Reject when `state` or `nonce` does not match stored values.
- Reject if `code` is missing or reused.
- Reject if ID token `iss`, `aud`, `exp`, or `nonce` is invalid.

Keep this stub small and reuse across providers.
