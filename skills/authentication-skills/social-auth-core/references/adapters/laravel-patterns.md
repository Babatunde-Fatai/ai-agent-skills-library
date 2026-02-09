# Laravel OAuth Patterns

**When to read:** When implementing social auth in Laravel.
**What problem it solves:** Laravel-specific routing, session, and redirect patterns for OAuth.
**When to skip:** If you are not using Laravel.
**Prerequisites:** Read `references/AGENT_EXECUTION_SPEC.md` and `references/patterns/oauth-flow-core.md`.

## Routes
Define login start and callback routes in `routes/web.php` to ensure session middleware.

Example:
```php
use Illuminate\Support\Facades\Route;

Route::get('/auth/{provider}', [SocialAuthController::class, 'start']);
Route::get('/auth/{provider}/callback', [SocialAuthController::class, 'callback']);
```

## Login Start (Server-Side)
- Generate `state`, `nonce`, and `code_verifier` on the server.
- Store them in the session or an encrypted cookie with a short TTL.
- Redirect the user to the provider authorization URL.

## Callback
- Read `code` and `state` from the query.
- Load and validate pre-auth state (one-time use, TTL enforced).
- Exchange `code` + `code_verifier` for tokens.
- Validate ID token if OIDC.
- Link or create user, rotate session, redirect.

## Session and Cookies
- Use a server-side session driver or encrypted cookies for pre-auth state.
- Keep Laravel's cookie encryption middleware enabled.

## Checklist
- [ ] Routes live in `routes/web.php` and use `web` middleware.
- [ ] Pre-auth state stored server-side or encrypted cookie with short TTL.
- [ ] State, nonce, and PKCE verified on callback.
- [ ] Tokens stored server-side only and encrypted at rest.
- [ ] Session ID rotated after login.
