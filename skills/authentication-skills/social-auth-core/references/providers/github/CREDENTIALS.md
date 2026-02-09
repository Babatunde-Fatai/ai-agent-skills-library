# GitHub OAuth Credentials Translation

## What GitHub Gives You

In GitHub Developer Settings -> OAuth Apps, you see:
- Client ID
- Client Secret
- Authorization callback URL (redirect URI)

GitHub does not provide a JSON download by default. Use the dashboard fields.

## What Your Backend Needs

## Session Secret (Required Before Proceeding)

Generate a secure session secret using: `openssl rand -base64 32`.
Add it to your environment variables as `SOCIAL_AUTH_SESSION_SECRET`.
Do this before continuing with the rest of the credentials setup.

Map the dashboard fields to these environment variables:

| Environment Variable | Credential Field | Example Value | Notes |
|---------------------|------------------|---------------|-------|
| `SOCIAL_AUTH_GITHUB_CLIENT_ID` | OAuth App Client ID | `Iv1.abc123def456` | Copy exactly |
| `SOCIAL_AUTH_GITHUB_CLIENT_SECRET` | OAuth App Client Secret | `gho_example_secret` | Keep secret |
| `SOCIAL_AUTH_GITHUB_REDIRECT_URI` | Not in dashboard export | `https://api.example.com/auth/github/callback` | Must match callback URL |
| `SOCIAL_AUTH_GITHUB_SCOPES` | Not in dashboard export | `read:user user:email` | Minimal login scopes |
| `SOCIAL_AUTH_SESSION_SECRET` | Not in dashboard export | Generate random 32+ bytes | Use the generated secret above |
| `FRONTEND_URL` | Not in dashboard export | `https://www.example.com` | Your frontend domain |

## IMPORTANT: Redirect URI Confusion

GitHub calls this the "Authorization callback URL" in the OAuth App settings.
Your backend env var must point to the backend callback endpoint, not the frontend.

Example:
```bash
SOCIAL_AUTH_GITHUB_REDIRECT_URI=https://api.example.com/auth/github/callback
```

This must exactly match the callback URL configured in GitHub.

## Quick Setup Checklist

- [ ] Copy Client ID to `SOCIAL_AUTH_GITHUB_CLIENT_ID`
- [ ] Copy Client Secret to `SOCIAL_AUTH_GITHUB_CLIENT_SECRET`
- [ ] Set `SOCIAL_AUTH_GITHUB_REDIRECT_URI` to `{backend_url}/auth/github/callback`
- [ ] Add that exact callback URL in GitHub OAuth App settings
- [ ] Set `SOCIAL_AUTH_GITHUB_SCOPES=read:user user:email`
- [ ] Set `SOCIAL_AUTH_SESSION_SECRET` from generated secret (required before proceeding)
- [ ] Set `FRONTEND_URL` to your frontend's domain

## Verification

Test your setup:
```bash
https://github.com/login/oauth/authorize?client_id=YOUR_CLIENT_ID&redirect_uri=YOUR_BACKEND_REDIRECT_URI&scope=read:user%20user:email
```

If redirect URI is correct, GitHub shows the authorization screen.
If wrong, GitHub returns a redirect URI error.
