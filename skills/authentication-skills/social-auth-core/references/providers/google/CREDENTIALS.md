# Google OAuth Credentials Translation

## What Google Gives You

When you create OAuth credentials in Google Cloud Console, you download a JSON file:
```json
{
  "web": {
    "client_id": "209914914434-abc123.apps.googleusercontent.com",
    "project_id": "my-project-12345",
    "auth_uri": "https://accounts.google.com/o/oauth2/auth",
    "token_uri": "https://oauth2.googleapis.com/token",
    "auth_provider_x509_cert_url": "https://www.googleapis.com/oauth2/v1/certs",
    "client_secret": "GOCSPX-Jqk_example_secret",
    "redirect_uris": ["https://www.example.com/callback"],
    "javascript_origins": ["https://www.example.com"]
  }
}
```

## What Your Backend Needs

## Session Secret (Required Before Proceeding)

Generate a secure session secret using: `openssl rand -base64 32`.
Add it to your environment variables as `SOCIAL_AUTH_SESSION_SECRET`.
Do this before continuing with the rest of the credentials setup.

Map the JSON fields to these environment variables:

| Environment Variable | JSON Path | Example Value | Notes |
|---------------------|-----------|---------------|-------|
| `SOCIAL_AUTH_GOOGLE_CLIENT_ID` | `web.client_id` | `209914914434-abc123.apps.googleusercontent.com` | Copy exactly |
| `SOCIAL_AUTH_GOOGLE_CLIENT_SECRET` | `web.client_secret` | `GOCSPX-Jqk_example_secret` | Keep secret |
| `SOCIAL_AUTH_GOOGLE_REDIRECT_URI` | NOT in JSON | `https://api.yourdomain.com/auth/google/callback` | See below |
| `SOCIAL_AUTH_GOOGLE_SCOPES` | Not in JSON | `openid email profile` | Standard for login |
| `SOCIAL_AUTH_SESSION_SECRET` | Not in JSON | Generate random 32+ bytes | Use the generated secret above |
| `FRONTEND_URL` | Not in JSON | `https://yourdomain.com` | Your frontend domain |

## IMPORTANT: Redirect URI Confusion

The `redirect_uris` in Google's JSON is NOT what you use for `SOCIAL_AUTH_GOOGLE_REDIRECT_URI`.

### Google JSON redirect_uris
```json
"redirect_uris": ["https://www.example.com/callback"]
```
- This is what you configured in Google Console
- Could be frontend or backend (depends on your setup)

### Your Backend Env Var
```bash
SOCIAL_AUTH_GOOGLE_REDIRECT_URI=https://api.example.com/auth/google/callback
```
- This MUST be your backend API endpoint
- Must exactly match one of the URIs you added to Google Console
- Google will send OAuth responses here

### What This Means

1. If you have a separate backend and frontend:
   - Backend deployed at: `https://api.example.com`
   - Frontend deployed at: `https://www.example.com`
   - Redirect URI should be: `https://api.example.com/auth/google/callback`
   - Add this to Google Console's authorized redirect URIs

2. If your backend is on a different domain (e.g., Render, Railway):
   - Backend: `https://my-app-xyz.onrender.com`
   - Frontend: `https://www.example.com`
   - Redirect URI: `https://my-app-xyz.onrender.com/auth/google/callback`
   - Add this to Google Console

3. Common mistake:
```bash
# WRONG - using frontend URL
SOCIAL_AUTH_GOOGLE_REDIRECT_URI=https://www.example.com/callback

# CORRECT - using backend API URL
SOCIAL_AUTH_GOOGLE_REDIRECT_URI=https://api.example.com/auth/google/callback
```

## Quick Setup Checklist

- [ ] Copy `client_id` from JSON to `SOCIAL_AUTH_GOOGLE_CLIENT_ID`
- [ ] Copy `client_secret` from JSON to `SOCIAL_AUTH_GOOGLE_CLIENT_SECRET`
- [ ] Determine your backend's public URL (where your API is deployed)
- [ ] Set `SOCIAL_AUTH_GOOGLE_REDIRECT_URI` to `{backend_url}/auth/google/callback`
- [ ] Add that exact redirect URI to Google Console's authorized list
- [ ] Set `SOCIAL_AUTH_SESSION_SECRET` from generated secret (required before proceeding)
- [ ] Set `SOCIAL_AUTH_GOOGLE_SCOPES=openid email profile`
- [ ] Set `FRONTEND_URL` to your frontend's domain

## Verification

Test your setup:
```bash
curl "https://accounts.google.com/o/oauth2/v2/auth?\
client_id=YOUR_CLIENT_ID&\
redirect_uri=YOUR_BACKEND_REDIRECT_URI&\
response_type=code&\
scope=openid%20email%20profile"
```

If redirect URI is correct, Google shows the login screen.
If wrong, you get: `Error 400: redirect_uri_mismatch`
