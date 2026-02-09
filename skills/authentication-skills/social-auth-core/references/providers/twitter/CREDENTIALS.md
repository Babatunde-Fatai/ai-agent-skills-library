# Twitter/X OAuth Credentials Translation

## What Twitter/X Gives You

In the X Developer Portal, you see:
- Client ID
- Client Secret (for OAuth 2.0)
- Callback URL / Redirect URI

For OAuth 1.0a apps, you may see API Key and API Key Secret instead.

## What Your Backend Needs

## Session Secret (Required Before Proceeding)

Generate a secure session secret using: `openssl rand -base64 32`.
Add it to your environment variables as `SOCIAL_AUTH_SESSION_SECRET`.
Do this before continuing with the rest of the credentials setup.

Map the dashboard fields to these environment variables:

| Environment Variable | Credential Field | Example Value | Notes |
|---------------------|------------------|---------------|-------|
| `SOCIAL_AUTH_TWITTER_CLIENT_ID` | Client ID | `QWERTY123456` | OAuth 2.0 apps |
| `SOCIAL_AUTH_TWITTER_CLIENT_SECRET` | Client Secret | `twitter_example_secret` | OAuth 2.0 apps |
| `SOCIAL_AUTH_TWITTER_REDIRECT_URI` | Not in dashboard export | `https://api.example.com/auth/twitter/callback` | Must match callback URL |
| `SOCIAL_AUTH_TWITTER_SCOPES` | Not in dashboard export | `users.read` | Minimal identity scope |
| `SOCIAL_AUTH_SESSION_SECRET` | Not in dashboard export | Generate random 32+ bytes | Use the generated secret above |
| `FRONTEND_URL` | Not in dashboard export | `https://www.example.com` | Your frontend domain |

Note: If you need refresh tokens, add `offline.access` to scopes.

## IMPORTANT: Redirect URI Confusion

The redirect URI must point to your backend callback endpoint and must match the callback URL
configured in your app settings.

Example:
```bash
SOCIAL_AUTH_TWITTER_REDIRECT_URI=https://api.example.com/auth/twitter/callback
```

## Quick Setup Checklist

- [ ] Copy Client ID to `SOCIAL_AUTH_TWITTER_CLIENT_ID`
- [ ] Copy Client Secret to `SOCIAL_AUTH_TWITTER_CLIENT_SECRET`
- [ ] Set `SOCIAL_AUTH_TWITTER_REDIRECT_URI` to `{backend_url}/auth/twitter/callback`
- [ ] Add that exact URL in your app callback settings
- [ ] Set `SOCIAL_AUTH_TWITTER_SCOPES=users.read` (add `offline.access` if needed)
- [ ] Set `SOCIAL_AUTH_SESSION_SECRET` from generated secret (required before proceeding)
- [ ] Set `FRONTEND_URL` to your frontend's domain

## Verification

OAuth 2.0 for X requires PKCE. Use your backend login-start endpoint to verify the redirect URI
is accepted. If the redirect URI is wrong, X returns a redirect mismatch error.
