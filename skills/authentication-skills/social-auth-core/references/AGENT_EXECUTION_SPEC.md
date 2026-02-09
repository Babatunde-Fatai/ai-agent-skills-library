# Agent Execution Spec (Social Auth Core)

**When to read:** Before any design or code changes.
**What problem it solves:** Enforces execution order and security invariants.
**When to skip:** Never for social auth tasks.
**Prerequisites:** Start from SKILL.md.

## Quick Navigation (Read This First)

- **Security Contract**: [AGENT_EXECUTION_SPEC.md](AGENT_EXECUTION_SPEC.md) ⚠️ MANDATORY
- **Core Flow**: [oauth-flow-core.md](patterns/oauth-flow-core.md)
- **Choose Provider**: [providers/](providers/) (Google, GitHub, LinkedIn, etc.)
- **Choose Framework**: [adapters/](adapters/) (Next.js, Express, vanilla Node, Laravel, Django, Flask, Rails, Vue)
- **Governance** (all required): env-contract.md, error-edge-cases.md, account-linking.md, testing-validation.md


## Role Selection and Multi-Part Implementation

OAuth/OIDC social authentication requires BOTH frontend and backend components working together. Before writing any code, determine which role you are implementing.

### Full-Stack Implementation (Recommended)
If you have access to the entire codebase:
1. Implement backend routes first (login start, callback handler).
2. Then implement frontend integration (login button, redirect handling).
3. Use the Implementation Context Transfer pattern below.


### Frontend Role (Client-Side)
You are in this role if working on UI, buttons, or browser-based redirect handling.

You MUST implement:
- Login button or link that redirects to backend route (for example, `GET /auth/google`).
- Optional loading states while auth is in progress.
- Optional error display for failed auth attempts.

You MUST NOT:
- Generate state, nonce, or PKCE verifier in the browser.
- Store tokens in localStorage or cookies.
- Call provider token endpoints.
- Validate ID tokens.
- Create user sessions.

When integrating with a backend that uses cookie-based sessions:

- Configure all API calls with `credentials: 'include'` (fetch) or `withCredentials: true` (Axios).
- Derive auth state from the server (e.g., GET /auth/me), not from localStorage or URL params.
- Do not add, remove, or modify existing auth logic that is unrelated to the social login being implemented.

Output for Backend Team: Document the login button route and any expected success or error redirects.


### Backend Role (Server-Side Authority)
You are in this role if working on API routes, OAuth handlers, database logic, or session management.

You MUST implement:
- Login start route (generates state, nonce, PKCE, redirects to provider).
- Callback route (exchanges code, validates tokens, creates session).
- State, nonce, verifier storage (server-side session or encrypted cookie).
- Token validation and storage (server-side only, encrypted at rest).
- User account creation or linking.
- Session rotation after successful login.

You MUST NOT:
- Expose client secrets to frontend.
- Trust frontend-provided state or tokens.
- Skip state or nonce validation.

Required Output (Backend-Only Mode): Frontend Integration Guide
- Exact routes the frontend must call (e.g., `GET /auth/google`, `GET /auth/me`, `POST /auth/logout`).
- How auth state is delivered (cookie-based; no tokens in URLs or localStorage).
- Redirect behavior after callback (e.g., `FRONTEND_URL/auth/callback?success=true|false`).
- Required fetch/Axios configuration (e.g., `credentials: "include"`).
- Logout flow (server clears cookies; frontend calls logout endpoint).
- Frontend MUST NOT: extract tokens from URLs, store tokens in localStorage, validate tokens client-side.
- Frontend env vars required (e.g., `NEXT_PUBLIC_API_URL` or equivalent backend base URL).

### Implementation Context Transfer

If implementing backend first, provide this to the frontend developer:
```json
{
  "loginRoute": "GET /auth/{provider}",
  "callbackBehavior": "Redirects to FRONTEND_URL/auth/callback?success=true|false",
  "userHydrationRoute": "GET /auth/me",
  "logoutRoute": "POST /auth/logout",
  "cookieConfig": "HTTP-only, Secure, SameSite=<value based on domain topology>",
  "fetchConfig": "All API calls must include credentials: 'include'",
  "frontendMustNot": ["extract tokens from URLs", "store tokens in localStorage", "validate tokens client-side"],
  "frontendEnvVars": ["NEXT_PUBLIC_API_URL or equivalent backend base URL"],
  "providers": ["{provider}"],
  "notes": "Auth state is cookie-based. Frontend only needs to read success/failure flag and call /auth/me. Replace {provider} with actual provider (e.g., 'google', 'github')."
}
```



If implementing frontend first (rare, only when backend already exists), provide this to the backend developer:
```json
{
  "loginButtonAction": "window.location.href = '/auth/google'",
  "expectedBehavior": "User redirects to Google, then back to callback URL",
  "currentCallbackRoute": "/auth/google/callback",
  "notes": "Backend must implement callback handler per AGENT_EXECUTION_SPEC"
}
```

## Non-Interference Rule [EXTREMELY IMPORTANT]

When implementing social auth, do NOT modify, refactor, or rewrite any existing authentication logic, endpoints, middleware, or guards that were not part of the task. Implement the requested social auth flow alongside what exists. If something in the existing codebase appears insecure or incompatible, document it as a recommendation in the output but do not change it unless explicitly asked.

## Non-Negotiable Security Rules

There are 15 required rules below.

- Authorization Code flow only.
- PKCE required for public clients and strongly recommended for confidential clients.
- PKCE method must be `S256`.
- `state` required and validated on callback.
- `nonce` required for OIDC and validated against the ID token.
- Redirect URIs must be explicit allowlist entries with exact match.
- Validate OIDC ID tokens: signature, `iss`, `aud`, `exp`, `iat`, `nonce`, and `azp` when required.
- Store tokens server-side only, encrypted at rest.
- Rotate session identifiers after successful login.
- Implement refresh token rotation and revocation when supported by provider.
- Rate limit login start and callback routes.
- Never log auth codes, tokens, client secrets, or full provider responses.
- HTTPS only for all auth endpoints and callback routes.
- Least-privilege scopes only.
- Implicit flow is forbidden.

## Execution Order (Required)

1. Discovery report and codebase analysis.
2. Read `AGENT_EXECUTION_SPEC.md` and all governance references in `governance/`.
3. Verify provider documentation and current endpoints.
   - If provider docs are inaccessible, use the provider reference file as baseline and add a note: "Implementation based on reference file; verify against live provider docs when accessible"
4. Select provider, patterns, and adapter references.
5. Plan file changes and environment variables.
6. Implement with security checklist in view.
7. Document all integration and architecture decisions in a Decisions Log section of the output. Include: what was preserved from existing auth, what changed, SameSite policy chosen and why, token delivery method, account linking policy, and any provider-specific choices.
8. Provide testing checklist.

### 1. Discovery Report

**NEW: Credential Translation Step**

After detecting the provider (e.g., Google), immediately:

1. Ask user for credential format:
   - "Please share the credential file/JSON you received from {provider}."
   - "Or describe what fields you have (client_id, client_secret, etc.)"

2. Load the provider's CREDENTIALS.md:
   - Open `references/providers/{provider}/CREDENTIALS.md`
   - Use the translation table to map their fields to env vars

3. Collect backend public URL (if not already provided).

4. Confirm callback URL:
   - Show `{BACKEND_URL}/auth/{provider}/callback`
   - Require confirmation that it matches the OAuth app configuration
   - Require confirmation they are using the backend domain, not the frontend

5. Provide concrete .env snippet:
```bash
# Based on your Google credentials:
SOCIAL_AUTH_GOOGLE_CLIENT_ID=<extracted from their JSON>
SOCIAL_AUTH_GOOGLE_CLIENT_SECRET=<extracted from their JSON>
SOCIAL_AUTH_GOOGLE_REDIRECT_URI=<infer from their backend URL>
SOCIAL_AUTH_GOOGLE_SCOPES=openid email profile
SOCIAL_AUTH_SESSION_SECRET=<tell them to generate>
FRONTEND_URL=<their frontend domain>
```

6. Validate redirect URI architecture:
   - Warn if they're confusing frontend/backend URLs
   - Remind them to add redirect URI to provider console

**Output**: User has concrete env vars ready before any code is written.

### Frontend-Only Scope

If operating in Frontend Role with an already-deployed backend, the Discovery Report reduces to:
1. Backend API base URL and its env var name (e.g., NEXT_PUBLIC_API_URL).
2. Auth routes: login start, user hydration, logout.
3. Callback redirect URL and query params (e.g., ?success=true|false).
4. Session mechanism: cookies or response body tokens.
5. Skip credential translation, provider doc verification, session store details, and account linking (backend concerns).


### After Discovery: Use Architecture Router

Once Discovery Report is complete and provider is confirmed:
1. Load `references/providers/{provider}/CREDENTIALS.md` for env var translation
2. Use Quick Architecture Router in SKILL.md to find relevant patterns
3. Load framework adapter
4. Proceed with implementation plan

Testing recommendation: add minimal mock-based tests for state parameter validation (reject missing/mismatched state) and token exchange failure handling (provider error/timeout). Use mocks/fakes to simulate the token endpoint returning errors and assert the callback fails safely without creating sessions. Comprehensive integration tests against real providers are optional, but state validation tests are recommended for production to prevent CSRF regressions.



## Provider-Specific Safety Notes

- Apple: client secret is a signed JWT; OIDC `nonce` is required; verify issuer and JWKS from Apple discovery.
- Google: treat as OIDC; verify ID token signature and claims using Google discovery metadata.
- GitHub: user email may be absent; handle verified emails via the emails endpoint.
- LinkedIn: permissions require approval and may vary by app; confirm scopes in the app dashboard.
- Facebook: permissions and app review requirements are strict; do not proceed without official doc confirmation.
- Twitter/X: OAuth 2.0 Authorization Code with PKCE is preferred; OAuth 1.0a only if a required API does not support OAuth 2.0.

## High-Signal Prompt Scaffold (Deterministic)

Use this required structure before writing code:

```text
ROLE: Social Auth Security Integrator
TASK: Implement <PROVIDER(S)> login in <FRAMEWORK/RUNTIME>

1) Discovery Report
- Framework/runtime detection results
- Current auth/session system
- Existing provider integrations
- User/account model and linking rules
- Deployment domains and redirect URI constraints

2) Provider Docs Verification
- Links and versions checked
- Endpoints confirmed
- Required scopes confirmed
- Provider-specific constraints confirmed

3) Required Environment Variables
- List all secrets and public keys
- Indicate which are server-only
- Separate backend/server-only vs frontend/public and state who must provide them
- Use this minimal output template:
```text
Backend (server-only; provided by backend/infra):
- <ENV_NAME>=<value or placeholder>

Frontend (public; provided by frontend/app config):
- <ENV_NAME>=<value or placeholder>
```

4) Planned File Changes
- Files to add or modify
- Routes and handlers to create

5) Security Checklist Compliance
- Authorization Code + PKCE (S256)
- state and nonce validation
- redirect allowlist
- ID token validation (if OIDC)
- token storage and session rotation
- logging redaction
- rate limiting

6) Implementation Plan
- Step-by-step plan with dependencies

7) Testing Checklist
- Unit tests
- Integration tests
- Negative tests (state/nonce mismatch, invalid redirect, expired token)
```

## Iterative Checkpoint Format (Use for Incremental Changes)

Use this format when:
- The codebase already has a partial auth implementation.
- The task is incremental (adding a provider, fixing a specific issue, updating token handling).
- A full Discovery Report already exists from a prior session.

Checkpoint format:
```text
CHECKPOINT: <brief description>
PRIOR CONTEXT: <reference to previous Discovery Report or implementation>
CHANGE SCOPE: <files affected, what is changing>
SECURITY IMPACT: <which of the 15 rules are affected by this change>
COMPATIBILITY CHECK: <does this break existing auth flows?>
IMPLEMENTATION: <the actual changes>
UPDATED DECISIONS: <any new integration decisions made>
```

Use the full scaffold for greenfield implementations. Use the checkpoint format for iterative changes when a full Discovery Report already exists.

If any section cannot be completed, stop and ask for clarification.
