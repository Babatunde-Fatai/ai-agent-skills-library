# LinkedIn (OAuth 2.0 + OIDC)

**When to read:** When implementing LinkedIn login.
**What problem it solves:** LinkedIn-specific endpoints, scopes, and constraints.
**When to skip:** If you are not using LinkedIn.
**Prerequisites:** Read `references/AGENT_EXECUTION_SPEC.md` and `references/patterns/oauth-flow-core.md`.

## Required OAuth Endpoints

- Authorization: `https://www.linkedin.com/oauth/v2/authorization`
- Token: `https://www.linkedin.com/oauth/v2/accessToken`

## Required Scopes and Risk Tradeoffs

- OIDC scopes for Sign In with LinkedIn: `openid profile email`.
- Legacy profile and email permissions: `r_liteprofile` and `r_emailaddress`.

Request only the minimum scopes required for the feature.

## Provider-Specific Constraints

- Permissions are tied to products and may require approval.
- Verify scope availability in the LinkedIn app dashboard.

## Example Environment Variables

- `SOCIAL_AUTH_LINKEDIN_CLIENT_ID`
- `SOCIAL_AUTH_LINKEDIN_CLIENT_SECRET`
- `SOCIAL_AUTH_LINKEDIN_REDIRECT_URI`
- `SOCIAL_AUTH_LINKEDIN_SCOPES=openid profile email`

## Compliance and Review

- App review may be required for protected permissions.

## Security Notes

- Enforce exact redirect URI matching.
- Use OIDC ID token validation when requesting OIDC scopes.
