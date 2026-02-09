# Apple (OAuth 2.0 + OIDC)

**When to read:** When implementing Sign in with Apple.
**What problem it solves:** Apple-specific endpoints, JWT client secret, and constraints.
**When to skip:** If you are not using Apple.
**Prerequisites:** Read `references/AGENT_EXECUTION_SPEC.md` and `references/patterns/oauth-flow-core.md`.

## Required OAuth Endpoints

- Authorization: `https://appleid.apple.com/auth/authorize`
- Token: `https://appleid.apple.com/auth/token`
- JWKS: `https://appleid.apple.com/auth/keys`
- Issuer: `https://appleid.apple.com`

## Required Scopes and Risk Tradeoffs

- `email` for email address.
- `name` for name attributes (only returned on first consent).

Request only what you need; Apple returns name only on first authorization.

## Provider-Specific Constraints

- Client secret must be a signed JWT (Team ID, Key ID, and Private Key).
- OIDC `nonce` must be used and validated.

## Example Environment Variables

- `SOCIAL_AUTH_APPLE_CLIENT_ID`
- `SOCIAL_AUTH_APPLE_CLIENT_SECRET`
- `SOCIAL_AUTH_APPLE_REDIRECT_URI`
- `SOCIAL_AUTH_APPLE_SCOPES=email name`

## Compliance and Review

- App must be configured in Apple Developer portal.

## Security Notes

- Validate ID token using Apple JWKS.
- Treat missing `name` on subsequent logins as expected behavior.
