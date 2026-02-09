# Facebook OAuth Credentials Translation

## What Facebook Gives You

In Meta for Developers, you see:
- App ID
- App Secret
- Valid OAuth Redirect URIs (in Facebook Login settings)

Facebook does not provide a JSON download by default. Use the dashboard fields.

## What Your Backend Needs

## Session Secret (Required Before Proceeding)

Generate a secure session secret using: `openssl rand -base64 32`.
Add it to your environment variables as `SOCIAL_AUTH_SESSION_SECRET`.
Do this before continuing with the rest of the credentials setup.

Map the dashboard fields to these environment variables:

| Environment Variable | Credential Field | Example Value | Notes |
|---------------------|------------------|---------------|-------|
| `SOCIAL_AUTH_FACEBOOK_CLIENT_ID` | App ID | `123456789012345` | Copy exactly |
| `SOCIAL_AUTH_FACEBOOK_CLIENT_SECRET` | App Secret | `fb_example_secret` | Keep secret |
| `SOCIAL_AUTH_FACEBOOK_REDIRECT_URI` | Not in dashboard export | `https://api.example.com/auth/facebook/callback` | Must match valid OAuth redirect URI |
| `SOCIAL_AUTH_FACEBOOK_SCOPES` | Not in dashboard export | `public_profile email` | Minimal login scopes |
| `SOCIAL_AUTH_SESSION_SECRET` | Not in dashboard export | Generate random 32+ bytes | Use the generated secret above |
| `FRONTEND_URL` | Not in dashboard export | `https://www.example.com` | Your frontend domain |

## IMPORTANT: Redirect URI Confusion

The redirect URI must point to your backend callback endpoint and must match one of the
"Valid OAuth Redirect URIs" in your Facebook Login settings.

Example:
```bash
SOCIAL_AUTH_FACEBOOK_REDIRECT_URI=https://api.example.com/auth/facebook/callback
```

## Quick Setup Checklist

- [ ] Copy App ID to `SOCIAL_AUTH_FACEBOOK_CLIENT_ID`
- [ ] Copy App Secret to `SOCIAL_AUTH_FACEBOOK_CLIENT_SECRET`
- [ ] Set `SOCIAL_AUTH_FACEBOOK_REDIRECT_URI` to `{backend_url}/auth/facebook/callback`
- [ ] Add that exact URL to Facebook's valid OAuth redirect URIs
- [ ] Set `SOCIAL_AUTH_FACEBOOK_SCOPES=public_profile email`
- [ ] Set `SOCIAL_AUTH_SESSION_SECRET` from generated secret (required before proceeding)
- [ ] Set `FRONTEND_URL` to your frontend's domain

## Verification

Test your setup:
```bash
https://www.facebook.com/dialog/oauth?client_id=YOUR_CLIENT_ID&redirect_uri=YOUR_BACKEND_REDIRECT_URI&response_type=code&scope=public_profile%20email
```

If redirect URI is correct, Facebook shows the authorization screen.
If wrong, Facebook returns a redirect URI error.
