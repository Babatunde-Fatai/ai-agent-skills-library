# Refresh Token Lifecycle

**When to read:** When a provider issues refresh tokens or you enable long-lived sessions.
**What problem it solves:** Prevents refresh token replay and stale-session risk.
**When to skip:** If the provider does not issue refresh tokens or you do not store them.
**Prerequisites:** Read `references/patterns/token-management.md`.

## Lifecycle Steps (Concise)
1. **Store securely**: Save refresh tokens server-side only, encrypted at rest.
2. **Rotate on use**: When a refresh token is used, store the new refresh token and invalidate the old one.
3. **Detect reuse**: If an old refresh token is reused, revoke all related refresh tokens and force re-auth.
4. **Revoke on logout**: Call provider revocation (if supported) and delete local token records.
5. **Minimize scope**: Only request `offline.access` or equivalent when required.

## Minimal Update Rules
- Update refresh token atomically with access token and expiry metadata.
- Never fall back to an old refresh token once rotation occurs.
- Treat refresh failures as a signal to re-auth, not as a reason to keep retrying.
