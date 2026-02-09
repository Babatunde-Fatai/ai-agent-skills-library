# OAuth Flow Core (Authorization Code + PKCE)

**When to read:** Always when implementing OAuth/OIDC flow.
**What problem it solves:** Defines required authorization code + PKCE steps.
**When to skip:** Never for OAuth implementations.
**Prerequisites:** Read `references/AGENT_EXECUTION_SPEC.md`.

## Required Flow Steps

1. Generate `state`, `nonce`, and `code_verifier` per login attempt.
2. Derive `code_challenge` from `code_verifier` using `S256`.
3. Store `state`, `nonce`, and `code_verifier` server-side and bind to a pre-auth session (session store or encrypted cookie — see `session-handling.md`).
4. Redirect to provider authorization endpoint with `response_type=code`, `code_challenge`, `code_challenge_method=S256`, `state`, and `nonce` (OIDC).
5. On callback, verify `state` matches stored value and enforce one-time use.
6. Exchange `code` for tokens at provider token endpoint with the original `code_verifier`.
7. If OIDC, validate ID token signature and claims using provider JWKS.
8. Create or update user account and rotate session identifier.
9. Persist tokens server-side only and record token metadata.

## Error Handling Requirements

- Reject missing or mismatched `state`.
- Reject missing `code`.
- Reject expired or invalid ID tokens.
- Do not retry code exchange with the same `code`.
- Log only sanitized error codes, never tokens or secrets.

## Multi-Provider Considerations

- Namescope state and PKCE data by provider.
- Use provider-specific callback routes or include provider id in state.
- Store provider ids separately to avoid collisions.
