---
name: social-auth
description: Security control system for OAuth 2.0 and OpenID Connect social login. Enforces discovery, strict security invariants, and deterministic outputs before any code is written. Covers OAuth login, provider integration, auth code plus PKCE flow, token and session handling, account linking, multi provider auth, and framework adapters (Next.js, Express, Node). Triggers on login flows, callbacks, token exchange, refresh, cookies, and adapter setup.
---

## Quick Navigation

**⚠️ READ FIRST**: [AGENT_EXECUTION_SPEC.md](references/AGENT_EXECUTION_SPEC.md) - Security contract and execution order

- **Providers**: Google | GitHub | LinkedIn | Apple | Twitter/X
- **Patterns**: OAuth Flow | Golden Path | Token Management | Refresh Token Lifecycle | Sessions | Data Models  
- **Adapters**: Next.js | Express | Vanilla Node | Pseudocode | Laravel | Django | Flask | Rails | Vue
- **Governance** (mandatory): Env Contract | Error Handling | Account Linking | Testing


## Mandatory Start (No Code)

1. Produce a Discovery Report before any code or config changes.
2. Read mandatory references: `references/AGENT_EXECUTION_SPEC.md` and all governance references in `references/governance/`.
3. Confirm provider(s), framework, language, deployment environment, and session strategy. If any are missing or ambiguous, stop and ask.
4. Verify current provider documentation and security guidance (RFC 9700, RFC 7636, RFC 8252, OIDC Core) before implementation.

## Multi-Part Implementation Note

OAuth/OIDC requires coordination between frontend and backend. This skill assumes full-stack access. If implementing in parts:

Scenario 1: Backend-only access
- Implement login start and callback routes.
- Document the route URLs for the frontend team.
- Provide success and error redirect URLs.
- See "Role Selection and Multi-Part Implementation" in `references/AGENT_EXECUTION_SPEC.md`.

Scenario 2: Frontend-only access
- Implement a login button that calls an existing backend route.
- Do not implement OAuth flow logic client-side.
- Use the Frontend-Only Discovery Report in AGENT_EXECUTION_SPEC.md.
- If the backend uses cookie-based sessions, configure API calls with credentials included and derive auth state from the server.
- Do not modify existing auth logic unrelated to the social login being added.

Scenario 3: Full-stack access (recommended)
- Implement backend first (routes, handlers, validation).
- Then add frontend integration (button, redirects).
- Test end-to-end before deploying.

## Discovery Report Requirements (Required First Output)

Include these items, even if the user did not ask:
- Detected framework and runtime (or ambiguity that blocks detection).
- Existing auth stack, session store, user model, and account linking rules.
- Current OAuth providers already integrated and any existing callback routes.
- Deployment constraints (domains, redirect URIs, environment variable system, secrets manager).
- Security middleware already present (CSRF, rate limiting, logging redaction).

If any required item is unknown and cannot be inferred from the repo, stop and ask.


## Routing Logic

Provider routing:
- If provider is Google, read `references/providers/google.md`.
- If provider is GitHub, read `references/providers/github.md`.
- If provider is LinkedIn, read `references/providers/linkedin.md`.
- If provider is Apple, read `references/providers/apple.md`.
- If provider is Twitter/X, read `references/providers/twitter.md`.
- If provider not supported, create a new provider file under `references/providers/` before proceeding.
- [All require CREDENTIALS.md first] Google | GitHub | LinkedIn | Apple | Twitter/X

Pattern routing:
- Always read `references/patterns/oauth-flow-core.md`.
- If you need a canonical end-to-end example, read `references/patterns/golden-path.md`.
- If tokens are stored, refreshed, or revoked, read `references/patterns/token-management.md`.
- If refresh tokens are issued or rotated, read `references/patterns/refresh-token-lifecycle.md`.
- If sessions or cookies are used, read `references/patterns/session-handling.md`.
- If the project has existing authentication, read `references/patterns/existing-auth-integration.md`.
- If cookies are used across different domains, read `references/patterns/samesite-decision-tree.md`.
- If OIDC ID token validation is needed without heavy dependencies, read `references/patterns/jwks-validation-helper.md`.
- If database changes are needed, read `references/patterns/data-models.md`.

Adapter routing:
- If framework is Express, read `references/adapters/express-patterns.md`.
- If framework is Next.js, read `references/adapters/nextjs-patterns.md`.
- If framework is vanilla Node or custom server, read `references/adapters/vanilla-node-patterns.md`.
- If framework is Laravel, read `references/adapters/laravel-patterns.md`.
- If framework is Django, read `references/adapters/django-patterns.md`.
- If framework is Flask, read `references/adapters/flask-patterns.md`.
- If framework is Rails, read `references/adapters/rails-patterns.md`.
- If framework is Vue, read `references/adapters/vue-patterns.md` and select a backend adapter for server routes.
- If framework is unknown or not covered, read `references/adapters/pseudocode-patterns.md` and create a new adapter file.

Governance routing (mandatory in every task):
- `references/governance/env-contract.md`
- `references/governance/error-edge-cases.md`
- `references/governance/account-linking.md`
- `references/governance/testing-validation.md`


## Quick Architecture Router (Use After Discovery)

Once you've completed the Discovery Report and confirmed the provider, use this router to find relevant documentation quickly:

**Backend Architecture**:
- Separate API + frontend → Section 4.2 (JWT) or 4.3 (Sessions)
- Monolithic (Next.js, Rails, Django) → Section 4.3 + framework adapter
- Serverless → Section 4.4 (not yet implemented, use pseudocode patterns)

**Provider Documentation**:
- Confirmed provider → `references/providers/{provider}/CREDENTIALS.md` then `references/providers/{provider}.md`
- Multiple providers → Read multi-provider patterns first

**Account Linking**:
- Have `auth_identities` table → Section 5.1
- Only `users` table → Section 5.2

This router does NOT replace discovery. Always complete the Discovery Report first.



## Forbidden Practices (Explicitly Prohibited)

- Implicit flow or token response in front-channel redirects.
- Skipping `state` or `nonce` validation.
- Wildcard or open redirect URI matching.
- Storing access, refresh, or ID tokens in localStorage or client-side cookies.
- Logging auth codes, tokens, client secrets, or full provider responses.
- Using unverified email claims as proof of identity.
- Reusing sessions across login (no session rotation).
- Committing `.env` files or hardcoding secrets in logic.
- Using the `sub` (subject) claim as a display name (it is an ID, not a name).

## Stop Conditions (Must Ask Clarifying Questions)

Stop and ask before any implementation if any of these are missing:
- Provider client credentials and allowed redirect URIs.
- The app framework and runtime are ambiguous or conflicting.
- Session strategy is unclear (cookie vs server session store).
- Account linking rules are unknown (merge by email or provider id).
- The environment variable system or secrets manager is not defined.
- Secrets Rotation Strategy is undefined (e.g., how to handle old vs new client secrets during a key roll).

## Required Output Structure (Required)

See `references/AGENT_EXECUTION_SPEC.md` for the required output structure (source of truth). Summary:
1. Discovery Report
2. Provider Docs Verification
3. Required Environment Variables
4. Planned File Changes
5. Security Checklist Compliance
6. Implementation Plan
7. Testing Checklist

In Required Environment Variables, always separate backend/server-only vs frontend/public and state who must provide them.

If you cannot comply with the required order, stop and ask for clarification.

## Multi-Provider Invariant

Assume multiple providers will coexist. Design routes, database models, and session logic so adding a second provider does not break the first.

## Documentation Freshness

Provider files are guidance only. You must verify the latest provider docs before coding. If docs conflict, stop and ask.
