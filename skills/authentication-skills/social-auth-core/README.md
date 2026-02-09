# Social Auth Core Skill

A security-first AI agent skill for implementing OAuth 2.0 + OpenID Connect (OIDC) social login. Provider-agnostic, framework-agnostic, and designed to enforce a deterministic, safe execution order before any code is written.

## Supported Providers (Out of the Box)

- Google
- GitHub
- LinkedIn
- Apple
- Twitter/X

If you need another provider, follow the skill’s pattern and add a new provider reference under `references/providers/`.

## What This Skill Does

This skill provides structured guidance for agents implementing:

- **OAuth login (Auth Code + PKCE)** - `state`, `nonce` (OIDC), PKCE `S256`, strict redirect URI handling
- **Provider integration** - Google, GitHub, LinkedIn, Apple, Twitter/X (and a pattern for adding others)
- **Token exchange + validation** - server-side token handling, OIDC ID token validation with JWKS
- **Sessions + cookies** - secure cookie/session configuration, session rotation after login
- **Token lifecycle** - storage rules, refresh token rotation, revocation on logout (where supported)
- **Account linking** - multi-provider identities, verified-email linking policy, takeover protections
- **Framework adapters** - Next.js, Express, vanilla Node, plus backend patterns for Laravel/Django/Flask/Rails and Vue integration guidance

## Key Design Decisions

### Discovery-First (No Code Until Discovery Report)

OAuth/OIDC work fails most often due to missing context: callback URLs, domain topology, existing auth stack, and account linking rules. This skill requires a **Discovery Report** before any code/config changes and defines stop conditions when required inputs are missing.

### Authorization Code + PKCE Only

The skill forbids implicit flow and enforces a canonical “Auth Code + PKCE (S256)” path:

```
1. Generate state, nonce, code_verifier
2. Derive code_challenge (S256)
3. Store state/nonce/verifier server-side (bound to a pre-auth session)
4. Redirect to provider authorize endpoint
5. Callback: validate state (+ nonce for OIDC), one-time use
6. Exchange code for tokens using code_verifier
7. Validate ID token (OIDC): signature + claims
8. Create/link user, rotate session, persist tokens server-side only
```

### Server-Side Token Discipline

- No tokens in `localStorage`.
- No tokens in client-readable cookies.
- Encrypt tokens at rest if persisted.
- Never log auth codes, tokens, client secrets, or full provider responses.

### Account Linking Policy (Takeover-Resistant)

The skill includes an explicit, safe “trusted email” rule: auto-link by email only when both sides are verified and provider asserts `email_verified`. It also covers collision handling for provider IDs.

## Skill Structure

```
social-auth-core/
├── SKILL.md                              # Entry point - start here
└── references/
    ├── AGENT_EXECUTION_SPEC.md           # Security contract + required output order
    ├── providers/                        # Provider specifics (Google, GitHub, LinkedIn, Apple, Twitter/X)
    ├── patterns/                         # OAuth flow, sessions, tokens, refresh lifecycle, data models
    ├── adapters/                         # Next.js, Express, vanilla Node, plus other framework patterns
    └── governance/                       # Env contract, edge cases, account linking, testing/validation
```

## How Agents Should Use This Skill

1. **Read `SKILL.md`** - routing logic for providers, patterns, adapters, and governance
2. **Start with `references/AGENT_EXECUTION_SPEC.md`** - execution order + non-negotiable security rules
3. **Pick provider docs** - read the provider reference and verify against live provider docs
4. **Pick an adapter** - Next.js/Express/vanilla Node, or pseudocode patterns for unsupported frameworks
5. **Follow governance** - env naming, edge cases, account linking, and testing checklist are mandatory

## Security Standards (Non-Negotiable)

- Authorization Code flow only; implicit flow is forbidden
- PKCE required (method `S256`)
- `state` required and validated; enforce one-time use
- `nonce` required for OIDC and validated against ID token
- Exact-match redirect URI allowlist (no wildcards)
- Server-side token storage only; encrypt at rest when persisted
- Session rotation after successful login
- Least-privilege scopes; no scope broadening without approval
- Rate limit login start + callback routes

## Environment Setup (Recommended)

Core variables follow a provider-scoped naming discipline:

| Key | Scope | Purpose |
|-----|-------|---------|
| `SOCIAL_AUTH_<PROVIDER>_CLIENT_ID` | Server (and sometimes public) | OAuth client id |
| `SOCIAL_AUTH_<PROVIDER>_CLIENT_SECRET` | Server only | OAuth client secret |
| `SOCIAL_AUTH_<PROVIDER>_REDIRECT_URI` | Server only | Callback URL registered with provider |
| `SOCIAL_AUTH_<PROVIDER>_SCOPES` | Server only | Requested scopes (e.g., `openid email profile`) |
| `SOCIAL_AUTH_SESSION_SECRET` | Server only | Encrypts short-lived OAuth state storage |
| `FRONTEND_URL` | Server only | Used for post-callback redirects (if applicable) |

Provider credential formats differ; the provider reference files include mapping guidance.

## Testing Checklist (Minimum)

- Reject missing/mismatched `state` (CSRF protection)
- Reject missing `code` on callback
- Verify `nonce` for OIDC providers
- Ensure session id rotates on login
- Verify cookie flags: `HttpOnly`, `Secure`, and correct `SameSite`
- Confirm tokens are never returned to frontend by default

## Quality Score

- **Score**: 9/10
- **Strengths**: discovery-first workflow, strong security invariants, multi-provider/account-linking guidance, broad adapter coverage
- **Minor gaps**: more concrete end-to-end samples per framework/provider; serverless-first adapter

---

Built for AI coding agents implementing OAuth/OIDC social login safely. Works with Claude Code, OpenAI Codex, GitHub Copilot, Cursor, and any agent that can read markdown.

---
### Developer Safety Note
Authentication is a high-risk surface. Before shipping:
* Verify the provider’s latest docs (endpoints, scopes, PKCE requirements, token claims).
* Audit `state`/`nonce` validation and session rotation.
* Confirm tokens never reach the browser unless explicitly required by your architecture.
