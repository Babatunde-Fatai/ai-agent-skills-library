# Testing and Validation

**When to read:** When planning or writing tests.
**What problem it solves:** Ensures required negative tests and security checks.
**When to skip:** If no tests are being written (not recommended).
**Prerequisites:** Read `references/AGENT_EXECUTION_SPEC.md`.

## Required Tests (The Checklist)

- [ ] **State Mismatch:** Simulating a callback with invalid `state` MUST reject login (403/400).
- [ ] **Nonce Mismatch:** (OIDC) Invalid `nonce` in ID Token MUST reject login.
- [ ] **Token Failure:** If token endpoint fails, NO session is created.
- [ ] **Session Rotation:** Session ID changes after successful login.
- [ ] **Email Policy:** Unverified emails are handled according to Account Linking policy.

## Integration Testing Strategy (Mocking)

**Do not attempt to hit real provider endpoints in CI/CD.**

For a minimal reusable stub, see `references/governance/minimal-test-harness.md`.

### 1. Intercept and Mock
Use tools like `msw` (Mock Service Worker) or `nock` to intercept outgoing requests to the provider.

- **Mock Discovery (`/.well-known/...`):** Return a static JSON.
- **Mock Token Endpoint (`/token`):**
  - **Assert:** Request body contains `code`, `client_id`, `client_secret`, `redirect_uri`, and `code_verifier`.
  - **Respond:** `200 OK` with `{ access_token: "mock_at", id_token: "mock_id_token" }`.

### 2. Mocking ID Tokens
When mocking OIDC, you must generate a valid signed JWT.
- Use a local private key to sign the mock ID token.
- Expose the corresponding public key in your mocked JWKS endpoint.

## Security Validation

- **Negative Testing:** Explicitly write tests that send:
    - `state` that doesn't match the cookie.
    - `code` that has already been used (replay attack).
    - `redirect_uri` that is slightly malformed.
- **Log Inspection:** Verify that no raw tokens or client secrets appear in test logs.
- **Rate Limiting:** Verify that hitting the login route 50 times in 1 second triggers a 429.
