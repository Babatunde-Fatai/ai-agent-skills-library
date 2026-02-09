# Flask OAuth Patterns

**When to read:** When implementing social auth in Flask.
**What problem it solves:** Flask-specific routing and session handling for OAuth.
**When to skip:** If you are not using Flask.
**Prerequisites:** Read `references/AGENT_EXECUTION_SPEC.md` and `references/patterns/oauth-flow-core.md`.

## Routes
Define login start and callback routes with Flask decorators.

Example:
```python
from flask import Flask

app = Flask(__name__)

@app.route("/auth/<provider>")
def oauth_start(provider):
    ...

@app.route("/auth/<provider>/callback")
def oauth_callback(provider):
    ...
```

## Login Start (Server-Side)
- Generate `state`, `nonce`, and `code_verifier` on the server.
- Store pre-auth state in a server-side session store or encrypted cookie with short TTL.
- Redirect to provider authorization URL.

## Callback
- Read `code` and `state` from the query.
- Load and validate pre-auth state (one-time use, TTL enforced).
- Exchange `code` + `code_verifier` for tokens.
- Validate ID token if OIDC.
- Link or create user, rotate session, redirect.

## Session and Cookies
- Flask's default session cookie is signed but not encrypted.
- Do not store `nonce` or `code_verifier` in readable cookies. Use server-side sessions or encrypted cookies.

## Checklist
- [ ] Routes defined for start and callback.
- [ ] Pre-auth state stored server-side or encrypted cookie with short TTL.
- [ ] State, nonce, and PKCE verified on callback.
- [ ] Tokens stored server-side only and encrypted at rest.
- [ ] Session ID rotated after login.
