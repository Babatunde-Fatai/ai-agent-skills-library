# Golden Path (Canonical End-to-End)
**When to read:** When you want a single, concrete example of the full OAuth/OIDC flow.
**What problem it solves:** Removes ambiguity by showing the minimum full-stack flow in one place.
**When to skip:** If you already have a working flow or need only a specific sub-step.
**Prerequisites:** Read `references/AGENT_EXECUTION_SPEC.md` and `references/patterns/oauth-flow-core.md`.

## Scope
This example assumes Next.js App Router + Google. Adapt the same steps to other frameworks and providers.

## Minimal Flow
1. Create login start route that generates `state`, `nonce`, and `code_verifier`, stores them server-side, and redirects to the provider authorization URL.
2. Create callback route that validates `state`, exchanges `code` + `code_verifier` for tokens, and validates the ID token.
3. Link or create the user account, rotate the session, and set the session cookie.
4. Redirect to a success URL; use a safe error URL on failure.

## Minimal File Plan (Example)
1. `app/api/auth/google/route.ts`
2. `app/api/auth/google/callback/route.ts`
3. `lib/auth/preauth.ts` (state/nonce/verifier storage)
4. `lib/oauth/google.ts` (auth URL + token exchange)
5. `lib/auth/session.ts` (rotate + set session)

Use `references/adapters/nextjs-patterns.md` for route handler shape and `references/providers/google.md` for endpoints and scopes.
