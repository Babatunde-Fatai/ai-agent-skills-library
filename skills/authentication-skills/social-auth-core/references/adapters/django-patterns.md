# Django OAuth Patterns

**When to read:** When implementing social auth in Django.
**What problem it solves:** Django-specific URL routing, views, and session handling for OAuth.
**When to skip:** If you are not using Django.
**Prerequisites:** Read `references/AGENT_EXECUTION_SPEC.md` and `references/patterns/oauth-flow-core.md`.

## Routes
Define login start and callback routes in `urls.py`.

Example:
```python
from django.urls import path
from . import views

urlpatterns = [
    path("auth/<provider>/", views.oauth_start, name="oauth_start"),
    path("auth/<provider>/callback/", views.oauth_callback, name="oauth_callback"),
]
```

## Login Start (Server-Side)
- Generate `state`, `nonce`, and `code_verifier` on the server.
- Store in `request.session` or an encrypted cookie with a short TTL.
- Redirect to provider authorization URL.

## Callback
- Read `code` and `state` from the query.
- Load and validate pre-auth state (one-time use, TTL enforced).
- Exchange `code` + `code_verifier` for tokens.
- Validate ID token if OIDC.
- Link or create user, rotate session, redirect.

## Session and Cookies
- Ensure `SessionMiddleware` is enabled.
- Prefer server-side session storage for pre-auth state.

## Checklist
- [ ] URL routes are defined for start and callback.
- [ ] Pre-auth state stored server-side or encrypted cookie with short TTL.
- [ ] State, nonce, and PKCE verified on callback.
- [ ] Tokens stored server-side only and encrypted at rest.
- [ ] Session ID rotated after login.
