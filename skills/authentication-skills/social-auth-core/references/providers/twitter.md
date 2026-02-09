# Twitter/X (OAuth 2.0)

**When to read:** When implementing Twitter/X login.
**What problem it solves:** X-specific endpoints, scopes, and OAuth 2.0 vs 1.0a constraints.
**When to skip:** If you are not using Twitter/X.
**Prerequisites:** Read `references/AGENT_EXECUTION_SPEC.md` and `references/patterns/oauth-flow-core.md`.

## Required OAuth Endpoints

- Authorization: `https://twitter.com/i/oauth2/authorize`
- Token: `https://api.twitter.com/2/oauth2/token`

## Required Scopes and Risk Tradeoffs

- `users.read` for basic profile info.
- `tweet.read` if needed for reading tweets.
- `offline.access` for refresh tokens.

Avoid broad scopes unless explicitly required.

## Provider-Specific Constraints

- OAuth 2.0 Authorization Code with PKCE is standard.
- OAuth 1.0a only if a required API does not support OAuth 2.0.

## Example Environment Variables

- `SOCIAL_AUTH_TWITTER_CLIENT_ID`
- `SOCIAL_AUTH_TWITTER_CLIENT_SECRET`
- `SOCIAL_AUTH_TWITTER_REDIRECT_URI`
- `SOCIAL_AUTH_TWITTER_SCOPES=users.read offline.access`

## Compliance and Review

- App must be configured in the X Developer Portal.

## Security Notes

- Enforce exact redirect URI matching.
- Validate tokens and expiry before use.
