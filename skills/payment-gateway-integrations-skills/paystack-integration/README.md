# Paystack Integration Skill

A Claude Code skill for implementing Paystack payment processing for African markets (Nigeria, Ghana, Kenya, South Africa). Built for AI agents integrating payments, framework agnostic with Next.js and Express implementation guidelines.

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
- **Minor gaps**: Could add "Quick Start" section, subscription lifecycle flowchart

---

Built for Claude Code agents, usable by other AI agents implementing African payment infrastructure. Covers frontend and backend implementations at a go.

---
### Developer Safety Note
This skill handles financial transactions. Before moving to production:
* Always verify AI-generated code.
* Test end-to-end flows in Paystack Test Mode.
* Audit your webhook signature verification—never skip the raw body check.
