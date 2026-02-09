# Rails OAuth Patterns

**When to read:** When implementing social auth in Rails.
**What problem it solves:** Rails routing, controller, and session handling for OAuth.
**When to skip:** If you are not using Rails.
**Prerequisites:** Read `references/AGENT_EXECUTION_SPEC.md` and `references/patterns/oauth-flow-core.md`.

## Routes
Define login start and callback routes in `config/routes.rb`.

Example:
```ruby
Rails.application.routes.draw do
  get "auth/:provider", to: "social_auth#start"
  get "auth/:provider/callback", to: "social_auth#callback"
end
```

## Login Start (Server-Side)
- Generate `state`, `nonce`, and `code_verifier` on the server.
- Store pre-auth state in session or encrypted cookie with short TTL.
- Redirect to provider authorization URL.

## Callback
- Read `code` and `state` from the query.
- Load and validate pre-auth state (one-time use, TTL enforced).
- Exchange `code` + `code_verifier` for tokens.
- Validate ID token if OIDC.
- Link or create user, rotate session, redirect.

## Session and Cookies
- Rails cookies are signed and encrypted by default.
- Prefer server-side session storage if you need stricter TTL controls.

## Checklist
- [ ] Routes defined in `config/routes.rb` for start and callback.
- [ ] Pre-auth state stored server-side or encrypted cookie with short TTL.
- [ ] State, nonce, and PKCE verified on callback.
- [ ] Tokens stored server-side only and encrypted at rest.
- [ ] Session ID rotated after login.
