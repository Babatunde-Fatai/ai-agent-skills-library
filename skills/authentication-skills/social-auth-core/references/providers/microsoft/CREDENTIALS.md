# Microsoft OAuth Credentials Translation

## What Microsoft Gives You

In Azure App Registrations, you see:
- Application (client) ID
- Directory (tenant) ID
- Client Secret (value)
- Redirect URIs configured in the portal

Microsoft does not provide a JSON download by default. Use the dashboard fields.

## What Your Backend Needs

## Session Secret (Required Before Proceeding)

Generate a secure session secret using: `openssl rand -base64 32`.
Add it to your environment variables as `SOCIAL_AUTH_SESSION_SECRET`.
Do this before continuing with the rest of the credentials setup.

Map the dashboard fields to these environment variables:

| Environment Variable | Credential Field | Example Value | Notes |
|---------------------|------------------|---------------|-------|
| `SOCIAL_AUTH_MICROSOFT_CLIENT_ID` | Application (client) ID | `11111111-2222-3333-4444-555555555555` | Copy exactly |
| `SOCIAL_AUTH_MICROSOFT_TENANT_ID` | Directory (tenant) ID | `common` or `contoso.onmicrosoft.com` | Use tenant or `common` |
| `SOCIAL_AUTH_MICROSOFT_CLIENT_SECRET` | Client Secret (value) | `microsoft_example_secret` | Keep secret |
| `SOCIAL_AUTH_MICROSOFT_REDIRECT_URI` | Not in dashboard export | `https://api.example.com/auth/microsoft/callback` | Must match portal redirect URI |
| `SOCIAL_AUTH_MICROSOFT_SCOPES` | Not in dashboard export | `openid profile email` | Minimal OIDC scopes |
| `SOCIAL_AUTH_SESSION_SECRET` | Not in dashboard export | Generate random 32+ bytes | Use the generated secret above |
| `FRONTEND_URL` | Not in dashboard export | `https://www.example.com` | Your frontend domain |

## IMPORTANT: Redirect URI Confusion

The redirect URI must point to your backend callback endpoint and must match one of the
redirect URIs configured in Azure App Registration.

Example:
```bash
SOCIAL_AUTH_MICROSOFT_REDIRECT_URI=https://api.example.com/auth/microsoft/callback
```

## Quick Setup Checklist

- [ ] Copy Application (client) ID to `SOCIAL_AUTH_MICROSOFT_CLIENT_ID`
- [ ] Copy Directory (tenant) ID to `SOCIAL_AUTH_MICROSOFT_TENANT_ID`
- [ ] Copy Client Secret value to `SOCIAL_AUTH_MICROSOFT_CLIENT_SECRET`
- [ ] Set `SOCIAL_AUTH_MICROSOFT_REDIRECT_URI` to `{backend_url}/auth/microsoft/callback`
- [ ] Add that exact URL in Azure App Registration redirect URIs
- [ ] Set `SOCIAL_AUTH_MICROSOFT_SCOPES=openid profile email`
- [ ] Set `SOCIAL_AUTH_SESSION_SECRET` from generated secret (required before proceeding)
- [ ] Set `FRONTEND_URL` to your frontend's domain

## Verification

Test your setup:
```bash
https://login.microsoftonline.com/YOUR_TENANT_ID/oauth2/v2.0/authorize?client_id=YOUR_CLIENT_ID&redirect_uri=YOUR_BACKEND_REDIRECT_URI&response_type=code&scope=openid%20profile%20email
```

If redirect URI is correct, Microsoft shows the authorization screen.
If wrong, Microsoft returns a redirect URI error.
