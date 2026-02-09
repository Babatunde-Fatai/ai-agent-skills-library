# Error and Edge Case Handling

**When to read:** When implementing callback or error handling.
**What problem it solves:** Prevents unsafe error handling and replay.
**When to skip:** If you are not touching callback or error flows.
**Prerequisites:** Read `references/patterns/oauth-flow-core.md`.

## Common Error Cases

- User denies consent or closes the provider consent screen.
- Provider returns `error` on callback.
- Missing or mismatched `state` or `nonce`.
- Missing `code` parameter.
- Token endpoint returns `invalid_grant` or `invalid_client`.
- Clock skew causes token `exp` validation failure.
- Provider user profile lacks email or email is unverified.

## Required Behaviors

- Fail closed on any state or nonce mismatch.
- Surface user-friendly errors without leaking sensitive details.
- Never retry a code exchange with the same `code`.
- Do not create accounts on incomplete identity data without explicit policy.
- Record audit events without tokens or secrets.

## Abuse and Rate Limiting

- Rate limit login initiation and callback endpoints.
- Detect repeated state mismatches as potential CSRF abuse.
- Block or challenge high-frequency login attempts.
