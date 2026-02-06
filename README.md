# Useful AI Agent Skills

A collection of production-ready skills for AI coding agents. Built and tested with Claude Code, but designed to work with any AI agent — OpenAI Codex, GitHub Copilot, Cursor, Windsurf, or any tool that can read structured markdown instructions.

## Why Agent Skills?

AI coding agents are powerful, but they need domain context to implement things like payment gateways correctly. These skills give your agent the guardrails, security patterns, and implementation references it needs to ship production-quality code — regardless of which agent you use.

## Available Skills

| Skill | Coverage | Key Features |
|-------|----------|--------------|
| [Flutterwave Integration](payment-gateway-integrations-skills/flutterwave-integration/) | 34+ African countries | Mobile money, card, bank transfer, subscriptions |
| [Paystack Integration](payment-gateway-integrations-skills/paystack-integration/) | Nigeria, Ghana, Kenya, South Africa | Card, bank, USSD, subscriptions |
| [Social Auth Core](authentication-skills/social-auth-core/) | OAuth 2.0 + OIDC | Google, GitHub, LinkedIn, Apple, Twitter/X + PKCE, sessions, account linking |

> More skills coming soon — contributions welcome.

---

## Quick Start

### 1. Pick a skill

Browse the [Available Skills](#available-skills) table above and open the skill folder you need.

### 2. Point your agent at it

Every skill has a `SKILL.md` entry point. How you load it depends on your agent:

**Claude Code (recommended — built and tested here first)**
```bash
# Add as a project skill (persists across sessions)
cp -r payment-gateway-integrations-skills/paystack-integration ~/.claude/skills/paystack-integration

# Add another skill the same way
cp -r authentication-skills/social-auth-core ~/.claude/skills/social-auth-core

# Or reference directly in a prompt
# "Read payment-gateway-integrations-skills/paystack-integration/SKILL.md and implement Paystack payments for my Next.js app"
```

**OpenAI Codex / ChatGPT**
```
Point your agent to the folder containing SKILL.md and references, and tell it to follow instructions within. Codex will follow the decision tree and reference files just like Claude does. Reference SKILL.md in your prompt:

"Follow the instructions in SKILL.md to add Flutterwave payments. ...{You can add other codebase specific instructions or unique features you want}"
```

**GitHub Copilot / Cursor / Windsurf**
```
Add the skill folder to your project (or workspace) so the agent can
discover it. Reference SKILL.md in your prompt:

"Follow the instructions in SKILL.md to add Flutterwave payments. ...{You can add other codebase specific instructions or unique features you want}"
```

**Any other agent**
```
These skills are plain markdown — no proprietary format. Any agent that
can read files or accept text context can use them. Just point them to SKILL.md and its references/ folder for deeper implementation details.
```

### 3. Give it a task

Once your agent has the skill loaded, prompt it with what you need:

```
"Set up Paystack one-time payments in my Next.js app with webhook verification"

"Add Flutterwave mobile money payments for Kenya and Ghana"

"Implement subscription billing with Paystack"

"Implement Google social login (OAuth/OIDC) in my Next.js app using Auth Code + PKCE"
```

The skill's decision tree will route your agent to the right reference files automatically.

### 4. Verify before shipping

These skills handle high-risk surfaces (money and identity). Always:
- Test end-to-end in sandbox/test mode using the provided test credentials
- Audit webhook signature verification (payments)
- Confirm amount verification logic (payments)
- Audit `state`/`nonce` validation and session rotation (auth)
- Review the `AGENT_EXECUTION_SPEC.md` safety checklist

---

# Flutterwave Integration Skill

An AI agent skill for implementing Flutterwave payment processing across 34+ African countries. Framework agnostic with Next.js and Express implementation guidelines.

## What This Skill Does

This skill provides structured guidance for coding agents implementing:

- **One-time payments** - Card, bank transfer, USSD, mobile money
- **Mobile money** - M-Pesa (Kenya), MTN/Vodafone/Airtel (Ghana, Uganda), Francophone Africa (XOF/XAF)
- **Subscriptions** - Payment plans and recurring billing
- **Bank transfers & payouts** - Collection and disbursement
- **Webhook handling** - Two verification methods with security guarantees

## Key Design Decisions

### tx_ref Generation (User-Generated)

Unlike platforms like Paystack where the API can generate references, Flutterwave requires **you** to generate the `tx_ref` before calling the API:

```
1. Generate unique tx_ref
2. Store in DB with status=pending
3. Call Flutterwave /v3/payments
4. Redirect user to returned link
```

This ensures your database always knows about the transaction before Flutterwave does, preventing orphan webhook events.

### Two Webhook Verification Methods

The skill documents both approaches:

1. **Simple (verif-hash)** - Direct header comparison to your secret hash
2. **Secure (HMAC-SHA256)** - Cryptographic signature with timing-safe comparison

Both are production-valid. Use HMAC when you can access the raw request body.

### Pan-African Mobile Money Coverage

Comprehensive documentation for mobile money across regions:

| Region | Countries | Networks |
|--------|-----------|----------|
| East Africa | Kenya, Uganda, Tanzania, Rwanda | M-Pesa, MTN, Airtel |
| West Africa | Ghana, Nigeria | MTN, Vodafone, Airtel, USSD |
| Francophone | Senegal, Ivory Coast, Mali, Cameroon | Orange, MTN, Moov |

## Skill Structure

```
flutterwave-integration/
├── SKILL.md                              # Entry point - start here
└── references/
    ├── AGENT_EXECUTION_SPEC.md           # Safety contract (read first)
    ├── one-time-payments.md              # Initialize, verify, tokenization
    ├── webhooks.md                       # Both verification methods
    ├── mobile-money.md                   # M-Pesa, MTN, Airtel, Francophone
    ├── bank-transfers.md                 # Payouts, PWBT collection
    ├── subscriptions.md                  # Payment plans, recurring billing
    ├── nextjs-implementation.md          # Complete App Router guide
    ├── express-implementation.md         # Full MVC structure
    ├── implementation-patterns.md        # Framework-agnostic pseudocode
    ├── flutterwave-specific.md           # v3 vs v4, enckey, split payments
    ├── troubleshooting.md                # Common errors, debug checklist
    ├── database-example.md               # Schema examples
    └── testing-with-ngrok.md             # Local webhook testing
```

## How Agents Should Use This Skill

1. **Read SKILL.md** - Contains the decision tree for which reference file to read
2. **Start with AGENT_EXECUTION_SPEC.md** - Defines the 10 payment invariants that prevent fraud
3. **Pick framework guide** - `nextjs-implementation.md` or `express-implementation.md`
4. **Add specialized features** - Mobile money, subscriptions, etc.

## Security Standards (Non-Negotiable)

- SHA256 HMAC webhook verification with `crypto.timingSafeEqual()`
- Amount verification in smallest currency unit (kobo/pesewas/cents)
- Raw body handling for webhook signature verification
- Secret key isolation (backend only, never frontend)
- Idempotent webhook handlers (check if order already paid before processing)

## Test Credentials

| Type | Number | Details |
|------|--------|---------|
| Success Card | `5531886652142950` | CVV: 564, Expiry: 09/32, PIN: 3310, OTP: 12345 |
| Failed Card | `5258585922666506` | Insufficient funds |
| Ghana Mobile | `0551234987` | OTP: 123456 |

Keys format:
- Test: `FLWSECK_TEST-*` / `FLWPUBK_TEST-*`
- Live: `FLWSECK-*` / `FLWPUBK-*`

## Differences from Paystack Skill

| Aspect | Flutterwave | Paystack |
|--------|-------------|----------|
| Reference | `tx_ref` (user generates) | `reference` (API can generate) |
| Webhook header | `verif-hash` or `flutterwave-signature` | `x-paystack-signature` |
| Hash algorithm | SHA256 | SHA512 |
| Countries | 34+ African countries | 4 (NG, GH, ZA, KE) |
| Mobile money | Built-in per-country types | Single `mobile_money` channel |
| API version | v3 (stable) / v4 (beta) | Single version |

## Quality Score

- **Score**: 9/10
- **Strengths**: Clear security-first approach, comprehensive mobile money docs, excellent TypeScript types
- **Minor gaps**: Could add subscription lifecycle flowchart

---

Built for AI coding agents implementing African payment infrastructure. Works with Claude Code, OpenAI Codex, GitHub Copilot, Cursor, and any agent that reads markdown. Covers frontend and backend implementations.

=======================================================

# Paystack Integration Skill

An AI agent skill for implementing Paystack payment processing for African markets (Nigeria, Ghana, Kenya, South Africa). Framework agnostic with Next.js and Express implementation guidelines.

## What This Skill Does

This skill provides structured guidance for coding agents implementing:

- **One-time payments** - Card, bank transfer, USSD, QR, mobile money
- **Subscriptions** - Plans, recurring billing, charge authorization
- **Webhook handling** - SHA512 HMAC verification with security guarantees
- **Multi-currency** - NGN (kobo), GHS (pesewas), ZAR (cents), KES (cents)

## Key Design Decisions

### Reference Flow (Paystack-First)

Unlike some platforms where you generate references locally, Paystack can generate references for you. The skill enforces a **Paystack-first** pattern:

```
1. Call Paystack /transaction/initialize (omit reference)
2. Paystack returns unique reference
3. Store reference in DB with status=pending
4. Redirect user to authorization_url
```

This prevents orphan database records when API calls fail.

### SHA512 Webhook Verification

Paystack uses HMAC-SHA512 for webhook signatures. The skill documents:

- Raw body capture before JSON parsing
- Timing-safe comparison with `crypto.timingSafeEqual()`
- Proper header extraction (`x-paystack-signature`)

### Kobo Conversion

All amounts are in smallest currency unit. The skill enforces:
- NGN → kobo (×100)
- GHS → pesewas (×100)
- ZAR/KES → cents (×100)

Amount verification prevents underpayment fraud.

## Skill Structure

```
paystack-integration/
├── SKILL.md                              # Entry point - start here
└── references/
    ├── AGENT_EXECUTION_SPEC.md           # Safety contract (read first)
    ├── one-time-payments.md              # Initialize, verify, popup/redirect
    ├── webhooks.md                       # Signature verification, events
    ├── subscriptions.md                  # Plans, recurring, authorization
    ├── nextjs-implementation.md          # Complete App Router guide
    ├── express-implementation.md         # Full MVC structure
    ├── implementation-patterns.md        # Framework-agnostic pseudocode
    └── troubleshooting.md                # Common errors, debug checklist
```

## How Agents Should Use This Skill

1. **Read SKILL.md** - Contains the decision tree for which reference file to read
2. **Start with AGENT_EXECUTION_SPEC.md** - Defines the payment invariants that prevent fraud
3. **Pick framework guide** - `nextjs-implementation.md` or `express-implementation.md`
4. **Add specialized features** - Subscriptions, charge authorization, etc.

## Security Standards (Non-Negotiable)

- SHA512 HMAC webhook verification with `crypto.timingSafeEqual()`
- Amount verification in smallest currency unit (kobo/pesewas/cents)
- Raw body handling for webhook signature verification
- Secret key isolation (backend only, never frontend)
- Idempotent webhook handlers (check if order already paid before processing)

## Test Credentials

| Type | Number | Details |
|------|--------|---------|
| Success Card | `4084084084084081` | CVV: any 3 digits, Expiry: any future, PIN: 1234, OTP: 123456 |
| Insufficient Funds | `5078575078575078` | Same details as above |
| Declined | `4084080000005408` | Same details as above |

Keys format:
- Test: `sk_test_*` / `pk_test_*`
- Live: `sk_live_*` / `pk_live_*`

## Environment Setup

| Key | Scope | Purpose |
|-----|-------|---------|
| `PAYSTACK_SECRET_KEY` | Backend Only | Authorizing API requests and verifying webhooks |
| `PAYSTACK_PUBLIC_KEY` | Frontend/Mobile | Initializing Inline.js popup/redirect |

## Quality Score

- **Score**: 8.5/10
- **Strengths**: Clear security-first approach, production-ready examples, comprehensive types
- **Minor gaps**: Could add subscription lifecycle flowchart

---

Built for AI coding agents implementing African payment infrastructure. Works with Claude Code, OpenAI Codex, GitHub Copilot, Cursor, and any agent that reads markdown. Covers frontend and backend implementations.

---
### Developer Safety Note
These skills handle financial transactions. Before moving to production:
* Always verify AI-generated code regardless of which agent produced it.
* Test end-to-end flows in sandbox/test mode with the provided test credentials.
* Audit your webhook signature verification — never skip the raw body check.

---

# Social Auth Core Skill

A security-first AI agent skill for implementing OAuth 2.0 + OpenID Connect (OIDC) social login. Provider-agnostic, framework-agnostic, and designed to enforce a deterministic, safe execution order before any code is written.

## Supported Providers (Out of the Box)

- Google
- GitHub
- LinkedIn
- Apple
- Twitter/X

If you need another provider, this skill includes a clear pattern for adding a new provider file under `authentication-skills/social-auth-core/references/providers/`.

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
authentication-skills/social-auth-core/
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
