---
name: social-auth
description: Security control system for OAuth 2.0 and OpenID Connect social login. Enforces discovery, strict security invariants, and deterministic outputs before any code is written. Covers OAuth login, provider integration, auth code plus PKCE flow, token and session handling, account linking, multi provider auth, and framework adapters (Next.js, Express, Node). Triggers on login flows, callbacks, token exchange, refresh, cookies, and adapter setup.
---

## Purpose

This skill orchestrates secure implementation of OAuth 2.0 and OpenID Connect social authentication flows.

It is responsible for:

- discovery before implementation
- routing to the correct provider, pattern, adapter, and governance documentation
- enforcing skill-specific constraints for social authentication
- ensuring that social auth is implemented alongside existing application boundaries without weakening repository-wide rules

This skill extends the global repository rules defined in:

- `../../core/SECURITY_INVARIANTS.md`
- `../../core/EXECUTION_RULES.md`
- `../../core/STOP_CONDITIONS.md`
- `../../core/DECISION_MODEL.md`

These core documents are mandatory and remain the source of truth for global security, execution behaviour, stop conditions, and conflict resolution.

## Quick Navigation

**⚠️ SKILL-SPECIFIC READ FIRST (after core docs)**: `references/AGENT_EXECUTION_SPEC.md`

Global repository rules are defined in `../../core/` and must be read before skill-specific execution documents.

Supporting references:

- **Providers**: Google | GitHub | LinkedIn | Apple | Twitter/X
- **Patterns**: OAuth Flow | Golden Path | Token Management | Refresh Token Lifecycle | Sessions | Data Models
- **Adapters**: Next.js | Express | Vanilla Node | Pseudocode | Laravel | Django | Flask | Rails | Vue
- **Governance** (mandatory): Env Contract | Error Handling | Account Linking | Testing

## Mandatory Reading Order

Read in this order before implementation:

1. `../../core/SECURITY_INVARIANTS.md`
2. `../../core/EXECUTION_RULES.md`
3. `../../core/STOP_CONDITIONS.md`
4. `../../core/DECISION_MODEL.md`
5. `references/AGENT_EXECUTION_SPEC.md`
6. All required governance files in `references/governance/`
7. Required pattern files in `references/patterns/`
8. Required provider files in `references/providers/`
9. Required adapter files in `references/adapters/`

Examples and concrete implementation patterns must be read only after the rule-bearing documents above have been understood.

## Mandatory Start (No Code)

Before any code or config change:

1. Produce a Discovery Report.
2. Read `references/AGENT_EXECUTION_SPEC.md`.
3. Read all mandatory governance references in `references/governance/`.
4. Confirm provider(s), framework, runtime, deployment environment, callback ownership model, and session strategy.
5. Verify current provider documentation and security guidance before implementation.
6. If required discovery inputs are missing or ambiguous, stop and ask rather than guessing.

This skill inherits all global stop conditions from `../../core/STOP_CONDITIONS.md`.

## Skill-Specific Discovery Requirements

The Discovery Report for this skill must identify, at minimum:

- detected framework and runtime
- whether the implementation is frontend-only, backend-only, or full-stack
- existing auth stack and session model
- existing user model and account-linking strategy
- current OAuth providers already integrated
- any existing callback/login/logout/auth-me routes
- deployment topology, domains, and redirect URI constraints
- environment variable system and secrets manager
- current CSRF, rate limiting, and logging redaction middleware
- existing auth middleware, guards, or route protection patterns
- whether auth state is server-driven, token-driven, or hybrid
- whether the requested flow is OAuth-only or OIDC-based
- whether multiple social providers must coexist now or later

If any required item is unknown and cannot be safely inferred, stop and ask.

## Multi-Part Implementation Note

OAuth and OIDC social authentication often require coordination between frontend and backend. This skill assumes that the implementation path must be selected explicitly during discovery.

### Scenario 1: Backend-only access

Use this mode when only the backend or API layer is in scope.

Responsibilities:

- implement login start and callback routes
- document the exact backend routes for the frontend
- define success and error redirect behaviour
- define session delivery behaviour
- provide frontend integration guidance without implementing client-side auth logic

See `references/AGENT_EXECUTION_SPEC.md` for role-specific output requirements.

### Scenario 2: Frontend-only access

Use this mode only when a backend social auth flow already exists.

Responsibilities:

- implement login button or route trigger to the backend
- integrate with server-delivered auth state
- configure credentialed API requests when cookie-based auth is used
- avoid changes to unrelated existing auth logic

Do not implement OAuth flow logic in the browser.

### Scenario 3: Full-stack access (recommended)

Use this mode when both frontend and backend are available.

Recommended sequence:

1. implement backend routes, handlers, validation, and session logic first
2. integrate frontend button, redirects, and auth-state hydration second
3. validate end-to-end behaviour before deployment

## Routing Logic

### Provider routing

After the provider is confirmed:

- If provider is Google, read:
  - `references/providers/google/CREDENTIALS.md`
  - `references/providers/google.md`

- If provider is GitHub, read:
  - `references/providers/github/CREDENTIALS.md`
  - `references/providers/github.md`

- If provider is LinkedIn, read:
  - `references/providers/linkedin/CREDENTIALS.md`
  - `references/providers/linkedin.md`

- If provider is Apple, read:
  - `references/providers/apple/CREDENTIALS.md`
  - `references/providers/apple.md`

- If provider is Twitter/X, read:
  - `references/providers/twitter/CREDENTIALS.md`
  - `references/providers/twitter.md`

If a provider is requested but is not supported by both a provider guide and credential guidance, stop and surface the support gap before proceeding.

### Pattern routing

Always read:

- `references/patterns/oauth-flow-core.md`

Read additional patterns when relevant:

- canonical end-to-end example → `references/patterns/golden-path.md`
- token storage, refresh, revocation → `references/patterns/token-management.md`
- refresh token lifecycle → `references/patterns/refresh-token-lifecycle.md`
- sessions or cookies → `references/patterns/session-handling.md`
- existing auth system present → `references/patterns/existing-auth-integration.md`
- cross-domain cookies → `references/patterns/samesite-decision-tree.md`
- OIDC ID token validation support → `references/patterns/jwks-validation-helper.md`
- database changes required → `references/patterns/data-models.md`

### Adapter routing

After framework/runtime is confirmed:

- Express → `references/adapters/express-patterns.md`
- Next.js → `references/adapters/nextjs-patterns.md`
- vanilla Node / custom server → `references/adapters/vanilla-node-patterns.md`
- Laravel → `references/adapters/laravel-patterns.md`
- Django → `references/adapters/django-patterns.md`
- Flask → `references/adapters/flask-patterns.md`
- Rails → `references/adapters/rails-patterns.md`
- Vue → `references/adapters/vue-patterns.md` plus an appropriate backend adapter
- unknown or unsupported framework → `references/adapters/pseudocode-patterns.md` and stop to surface adapter limitations where needed

### Governance routing (mandatory for every task)

Always read:

- `references/governance/env-contract.md`
- `references/governance/error-edge-cases.md`
- `references/governance/account-linking.md`
- `references/governance/testing-validation.md`

## Quick Architecture Router (Use After Discovery)

This router is a convenience layer and does not replace discovery.

### Backend architecture

- Separate API + frontend → evaluate token/session architecture and relevant adapter patterns
- Monolithic application (for example Next.js, Rails, Django) → use session-oriented patterns plus relevant framework adapter
- Serverless → use pseudocode patterns if no dedicated adapter exists and surface any unresolved runtime constraints

### Provider documentation

- Confirmed provider → read `references/providers/{provider}/CREDENTIALS.md` first, then `references/providers/{provider}.md`
- Multiple providers → read provider docs for each confirmed provider and ensure multi-provider compatibility from the beginning

### Account linking

- Existing identity-linking table or strategy → align implementation with that design
- Only a users table exists → consult data-model and account-linking governance docs before proposing schema changes

## Skill-Specific Constraints

This skill adds the following scoped constraints on top of the global core rules:

- Authorization Code flow is required for OAuth/OIDC social login.
- PKCE must be used where required and should default to `S256`.
- `state` validation is mandatory.
- `nonce` validation is mandatory for OIDC flows.
- Session rotation is required after successful login.
- Social auth must be integrated without rewriting unrelated existing auth logic unless explicitly requested.
- Account linking rules must be explicit before implementation.
- Routes, models, and session logic must be designed so that multiple providers can coexist safely.

These constraints are enforced in detail by `references/AGENT_EXECUTION_SPEC.md`.

## Social Auth Forbidden Practices

In addition to global security invariants, the following are explicitly prohibited for this skill:

- implicit flow or token response in front-channel redirects
- skipping `state` validation
- skipping `nonce` validation for OIDC
- wildcard or open redirect URI matching
- storing access, refresh, or ID tokens in localStorage
- logging auth codes, tokens, client secrets, or full provider responses
- using unverified email claims as proof of identity
- reusing pre-login session identifiers after successful auth
- hardcoding secrets in source or config tracked by version control
- using the `sub` claim as a display name

## Required Output Structure

The required output structure for this skill is defined in:

- `references/AGENT_EXECUTION_SPEC.md`

At minimum, output should remain structured around:

1. Discovery Report
2. Provider Docs Verification
3. Required Environment Variables
4. Planned File Changes
5. Security Checklist Compliance
6. Implementation Plan
7. Testing Checklist

When listing environment variables, always separate:

- backend/server-only values
- frontend/public values

and clearly state ownership.

## Multi-Provider Invariant

Assume more than one provider may need to coexist.

Design:

- routes
- data model changes
- identity linking
- session logic
- provider abstractions

so that adding a second provider does not require breaking the first.

## Documentation Freshness

Provider files in this repository are implementation guidance, not a guarantee of current provider documentation.

Before coding:

- verify current provider docs
- verify current endpoints and requirements
- verify scopes and callback expectations
- surface conflicts if repository guidance and live provider docs differ
- treat live provider requirements as authoritative when they introduce stricter or updated provider-specific constraints

If provider support is partial, outdated, or ambiguous, stop and ask before implementation.
