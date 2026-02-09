# Flutterwave Integration Skill

A Claude Code skill for implementing Flutterwave payment processing across 34+ African countries. Built for agents integrating payments, framework agnostic with Next.js and Express implementation guidelines.

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
- **Minor gaps**: Could add "Quick Start" section, subscription lifecycle flowchart

---

Built for Claude Code agents, usable by other AI agents implementing African payment infrastructure. Covers frontend and backend implementations at a go.

---
### ⚠️ Developer Safety Note
This skill handles financial transactions. Before moving to production:
* Always verify AI-generated code.
* Test end-to-end flows in Flutterwave Test Mode.
* Audit your webhook signature verification—never skip the raw body check.
