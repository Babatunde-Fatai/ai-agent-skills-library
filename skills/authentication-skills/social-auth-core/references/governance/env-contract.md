# Environment Variable Contract

**When to read:** When defining or reviewing environment variables and secrets.
**What problem it solves:** Standardizes env naming and server-only scope.
**When to skip:** If no configuration changes are needed.
**Prerequisites:** Read `references/AGENT_EXECUTION_SPEC.md`.

## Naming Discipline

- Use consistent, provider-scoped names.
- Keep server-only secrets separate from public config.
- Do not reuse variable names across providers.

## Recommended Naming Pattern

- `SOCIAL_AUTH_<PROVIDER>_CLIENT_ID`
- `SOCIAL_AUTH_<PROVIDER>_CLIENT_SECRET`
- `SOCIAL_AUTH_<PROVIDER>_REDIRECT_URI`
- `SOCIAL_AUTH_<PROVIDER>_SCOPES`
- `SOCIAL_AUTH_SESSION_SECRET`

## Rules

- `SOCIAL_AUTH_SESSION_SECRET` encrypts the short-lived OAuth state cookie (state, nonce, PKCE verifier). It is server-only and should be a random 32+ byte string. It is NOT related to user sessions or JWTs. Generate with `openssl rand -base64 32`.
- Client secrets are server-only and must never be exposed to the browser.
- Redirect URIs must match provider allowlist exactly.
- Document any required public variables separately and mark them safe to expose.
