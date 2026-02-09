# Apple OAuth Credentials Translation

## What Apple Gives You

For Sign in with Apple (web), you create:
- Service ID (this is your Client ID)
- Key ID
- Team ID
- Private key file (.p8) downloaded from Apple Developer
- Return URLs configured in the Service ID settings

Apple does not provide a JSON download. You must map the dashboard fields and the .p8 file.

## What Your Backend Needs

## Session Secret (Required Before Proceeding)

Generate a secure session secret using: `openssl rand -base64 32`.
Add it to your environment variables as `SOCIAL_AUTH_SESSION_SECRET`.
Do this before continuing with the rest of the credentials setup.

Map the dashboard fields to these environment variables:

| Environment Variable | Credential Field | Example Value | Notes |
|---------------------|------------------|---------------|-------|
| `SOCIAL_AUTH_APPLE_CLIENT_ID` | Service ID | `com.example.web` | Copy exactly |
| `SOCIAL_AUTH_APPLE_TEAM_ID` | Team ID | `ABCD1234EF` | Required for client secret JWT |
| `SOCIAL_AUTH_APPLE_KEY_ID` | Key ID | `1A2B3C4D5E` | Required for client secret JWT |
| `SOCIAL_AUTH_APPLE_PRIVATE_KEY` | .p8 private key contents | `-----BEGIN PRIVATE KEY-----...` | Store securely |
| `SOCIAL_AUTH_APPLE_REDIRECT_URI` | Not in dashboard export | `https://api.example.com/auth/apple/callback` | Must match Return URL |
| `SOCIAL_AUTH_APPLE_SCOPES` | Not in dashboard export | `name email` | Apple scopes |
| `SOCIAL_AUTH_SESSION_SECRET` | Not in dashboard export | Generate random 32+ bytes | Use the generated secret above |
| `FRONTEND_URL` | Not in dashboard export | `https://www.example.com` | Your frontend domain |

Note: `SOCIAL_AUTH_APPLE_PRIVATE_KEY` often needs newlines preserved. If stored in env vars,
replace line breaks with `\n` and restore them at runtime.

## IMPORTANT: Redirect URI Confusion

The redirect URI must point to your backend callback endpoint and must match one of the
"Return URLs" configured in your Apple Service ID settings.

Example:
```bash
SOCIAL_AUTH_APPLE_REDIRECT_URI=https://api.example.com/auth/apple/callback
```

## Quick Setup Checklist

- [ ] Copy Service ID to `SOCIAL_AUTH_APPLE_CLIENT_ID`
- [ ] Copy Team ID to `SOCIAL_AUTH_APPLE_TEAM_ID`
- [ ] Copy Key ID to `SOCIAL_AUTH_APPLE_KEY_ID`
- [ ] Store .p8 contents in `SOCIAL_AUTH_APPLE_PRIVATE_KEY`
- [ ] Set `SOCIAL_AUTH_APPLE_REDIRECT_URI` to `{backend_url}/auth/apple/callback`
- [ ] Add that exact URL to Apple Service ID Return URLs
- [ ] Set `SOCIAL_AUTH_APPLE_SCOPES=name email`
- [ ] Set `SOCIAL_AUTH_SESSION_SECRET` from generated secret (required before proceeding)
- [ ] Set `FRONTEND_URL` to your frontend's domain

## Verification

Test your setup:
```bash
https://appleid.apple.com/auth/authorize?client_id=YOUR_CLIENT_ID&redirect_uri=YOUR_BACKEND_REDIRECT_URI&response_type=code&scope=name%20email
```

If redirect URI is correct, Apple shows the authorization screen.
If wrong, Apple returns a redirect URI error.
