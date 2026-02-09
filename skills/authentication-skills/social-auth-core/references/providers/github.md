# GitHub (OAuth 2.0)

**When to read:** When implementing GitHub login.
**What problem it solves:** GitHub-specific endpoints, scopes, and constraints.
**When to skip:** If you are not using GitHub.
**Prerequisites:** Read `references/AGENT_EXECUTION_SPEC.md` and `references/patterns/oauth-flow-core.md`.

## Required OAuth Endpoints

- Authorization: `https://github.com/login/oauth/authorize`
- Token: `https://github.com/login/oauth/access_token`

## Required Scopes and Risk Tradeoffs

- `read:user` for basic profile access.
- `user:email` to read email addresses because primary email may be private.

Avoid broad scopes such as `repo` unless a feature requires it.

## Provider-Specific Constraints

- Implicit flow is not supported.
- PKCE is required for public clients.

## Example Environment Variables

- `SOCIAL_AUTH_GITHUB_CLIENT_ID`
- `SOCIAL_AUTH_GITHUB_CLIENT_SECRET`
- `SOCIAL_AUTH_GITHUB_REDIRECT_URI`
- `SOCIAL_AUTH_GITHUB_SCOPES=read:user user:email`

## Compliance and Review

- App must be registered in GitHub Developer Settings.

## Security Notes

- Use the `/user/emails` API to obtain verified email addresses.
- Enforce exact redirect URI matching.
