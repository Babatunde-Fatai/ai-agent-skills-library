# Token Management

**When to read:** When storing, refreshing, or revoking tokens.
**What problem it solves:** Safe token storage and rotation rules.
**When to skip:** If tokens are not stored or refreshed.
**Prerequisites:** Read `references/patterns/oauth-flow-core.md`.

## Storage Rules

- Store tokens server-side only.
- Encrypt tokens at rest using application-level encryption.
- Store token metadata separately: provider, scopes, issued_at, expires_at, refresh_expires_at.
- Never store tokens in localStorage or client-readable cookies.

## Refresh and Rotation

- Use refresh token rotation when the provider supports it.
- Invalidate or revoke the previous refresh token after rotation.
- Detect refresh token reuse and force re-auth.
- Do not refresh on every request; refresh only when within a short expiry window.
 - For lifecycle steps, see `references/patterns/refresh-token-lifecycle.md`.

## Revocation and Logout

- On logout, revoke refresh tokens if the provider supports revocation.
- Clear server-side session and remove token records.
- Treat revocation failure as a warning, not a success.

## Scope and Audience Discipline

- Record exact scopes granted and compare with requested scopes.
- Fail closed if scope set is narrower than required for the feature.
- Never broaden scopes without explicit user approval.
