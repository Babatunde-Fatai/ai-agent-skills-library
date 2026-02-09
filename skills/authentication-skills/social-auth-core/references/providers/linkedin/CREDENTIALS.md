# LinkedIn OAuth Credentials Translation

## What LinkedIn Gives You

In the LinkedIn Developer Portal, you see:
- Client ID
- Client Secret
- Authorized redirect URLs list

LinkedIn does not provide a JSON download by default. Use the dashboard fields.

## What Your Backend Needs

## Session Secret (Required Before Proceeding)

Generate a secure session secret using: `openssl rand -base64 32`.
Add it to your environment variables as `SOCIAL_AUTH_SESSION_SECRET`.
Do this before continuing with the rest of the credentials setup.

Map the dashboard fields to these environment variables:

| Environment Variable | Credential Field | Example Value | Notes |
|---------------------|------------------|---------------|-------|
| `SOCIAL_AUTH_LINKEDIN_CLIENT_ID` | Client ID | `86abc12def3456` | Copy exactly |
| `SOCIAL_AUTH_LINKEDIN_CLIENT_SECRET` | Client Secret | `linkedInExampleSecret` | Keep secret |
| `SOCIAL_AUTH_LINKEDIN_REDIRECT_URI` | Not in dashboard export | `https://api.example.com/auth/linkedin/callback` | Must match authorized redirect URL |
| `SOCIAL_AUTH_LINKEDIN_SCOPES` | Not in dashboard export | `openid profile email` | OIDC scopes (confirm in app) |
| `SOCIAL_AUTH_SESSION_SECRET` | Not in dashboard export | Generate random 32+ bytes | Use the generated secret above |
| `FRONTEND_URL` | Not in dashboard export | `https://www.example.com` | Your frontend domain |

Note: Some legacy LinkedIn apps use `r_liteprofile r_emailaddress` instead of OIDC scopes.

## IMPORTANT: Redirect URI Confusion

The redirect URI must point to your backend callback route and must match one of the
"Authorized redirect URLs" in LinkedIn.

Example:
```bash
SOCIAL_AUTH_LINKEDIN_REDIRECT_URI=https://api.example.com/auth/linkedin/callback
```

## Quick Setup Checklist

- [ ] Copy Client ID to `SOCIAL_AUTH_LINKEDIN_CLIENT_ID`
- [ ] Copy Client Secret to `SOCIAL_AUTH_LINKEDIN_CLIENT_SECRET`
- [ ] Set `SOCIAL_AUTH_LINKEDIN_REDIRECT_URI` to `{backend_url}/auth/linkedin/callback`
- [ ] Add that exact URL to LinkedIn's authorized redirect URLs
- [ ] Set `SOCIAL_AUTH_LINKEDIN_SCOPES=openid profile email` (or legacy scopes if required)
- [ ] Set `SOCIAL_AUTH_SESSION_SECRET` from generated secret (required before proceeding)
- [ ] Set `FRONTEND_URL` to your frontend's domain

## Verification

Test your setup:
```bash
https://www.linkedin.com/oauth/v2/authorization?response_type=code&client_id=YOUR_CLIENT_ID&redirect_uri=YOUR_BACKEND_REDIRECT_URI&scope=openid%20profile%20email
```

If redirect URI is correct, LinkedIn shows the authorization screen.
If wrong, LinkedIn returns a redirect URI error.
