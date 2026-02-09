# Google (OAuth 2.0 + OIDC)

**When to read:** When implementing Google login.
**What problem it solves:** Google-specific endpoints, scopes, and constraints.
**When to skip:** If you are not using Google.
**Prerequisites:** Read `references/AGENT_EXECUTION_SPEC.md` and `references/patterns/oauth-flow-core.md`.

## Required OAuth Endpoints

- Authorization: `https://accounts.google.com/o/oauth2/v2/auth`
- Token: `https://oauth2.googleapis.com/token`
- Discovery: `https://accounts.google.com/.well-known/openid-configuration`
- JWKS: `https://www.googleapis.com/oauth2/v3/certs`
- UserInfo: `https://openidconnect.googleapis.com/v1/userinfo`

## Required Scopes and Risk Tradeoffs

- `openid` for OIDC ID token issuance.
- `email` for verified email address.
- `profile` for basic profile data.

Avoid requesting additional Google API scopes unless explicitly required.

## Provider-Specific Constraints

- Use OIDC ID token verification with issuer and JWKS from discovery.
- Offline access requires explicit consent and may require `access_type=offline`.

## Example Environment Variables

- `SOCIAL_AUTH_GOOGLE_CLIENT_ID`
- `SOCIAL_AUTH_GOOGLE_CLIENT_SECRET`
- `SOCIAL_AUTH_GOOGLE_REDIRECT_URI`
- `SOCIAL_AUTH_GOOGLE_SCOPES=openid email profile`

## Compliance and Review

- OAuth consent screen configuration required.
- Sensitive or restricted scopes may require app verification.

## Security Notes

- Validate `iss`, `aud`, `exp`, `iat`, and `nonce` on ID tokens.
- Enforce exact redirect URI matching.
