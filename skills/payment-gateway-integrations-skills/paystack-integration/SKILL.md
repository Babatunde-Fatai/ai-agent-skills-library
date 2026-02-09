---
name: paystack-integration
description: >
  Implement Paystack payment processing for Nigerian and African markets. Use when integrating one-time payments (initialize, redirect/popup, verify), subscriptions/recurring billing (plans, customers, subscriptions), or webhook handling. Covers Next.js and Express implementations with TypeScript. Triggers: payment integration, Paystack setup, accept payments Nigeria, NGN payments, subscription billing, payment webhooks, Paystack API, kobo conversion.
---


# Paystack Integration

Paystack is Africa's leading payment gateway, primarily serving Nigeria, Ghana, Kenya, and South Africa. This skill covers complete payment integration patterns.

## Table of Contents (all contained in .references/)

0. [Agent Execution Spec](references/AGENT_EXECUTION_SPEC.md) ⚠️ READ FIRST
1. [Quick Reference](#quick-reference)
2. [Payment Flow Overview](#payment-flow-overview)
3. [Core Implementation](#core-implementation)
4. [Webhook Essentials](#webhook-essentials)
5. [Framework Guides](#framework-guides)
6. [Deployment Checklist](#deployment-checklist)
7. [Quick Troubleshooting](#quick-troubleshooting)

---
 
## File Selection Decision Tree (quick)

- If implementing frontend popup/redirect → read [references/one-time-payments.md](references/one-time-payments.md)
- If implementing backend verification → read [references/express-implementation.md](references/express-implementation.md)
- If implementing webhooks → read [references/webhooks.md](references/webhooks.md)
- If implementing subscriptions or charge authorization → read [references/subscriptions.md](references/subscriptions.md)
- If unsure about responsibilities or safety → read [references/AGENT_EXECUTION_SPEC.md](references/AGENT_EXECUTION_SPEC.md) first
- Need framework-specific examples → read [references/nextjs-implementation.md](references/nextjs-implementation.md) or [references/express-implementation.md](references/express-implementation.md)

## ⚠️ For Coding Agents: Start Here

Before reading any examples or framework guides, you MUST read:

**[Agent Execution Spec](references/AGENT_EXECUTION_SPEC.md)**

This file defines the payment safety contract and execution order that must be followed to prevent fraud and incorrect implementations.

Condensed Payment Safety Rules (summary — see execution spec for full invariants):

- Call Paystack `initializeTransaction` FIRST, then store the returned reference in DB.
- Always convert and store amounts in smallest currency unit (kobo/pesewa/cents) and compare those values.
- Never trust client-side success callbacks; always verify on backend using the reference.
- Webhook handlers must verify `x-paystack-signature` using the raw request body before parsing.
- Verify that the verified amount exactly matches the expected DB amount before fulfilling.
- Ensure idempotency: if order.status == 'paid', exit immediately.

Examples in this SKILL are references. The execution spec is the protocol.

---
## Frontend and Backend Integration Bridge

In most implementations, frontend and backend are separate or separated by folders.

After a successful popup or redirect, the frontend MUST send the transaction reference to the backend for verification.

Recommended pattern:

Frontend → POST /api/verify-payment { reference }

The backend then performs Paystack verification and marks the order as paid.

**Reference consistency is critical**: The reference returned from `initializeTransaction` is stored in your database and sent to the frontend. This same reference appears in webhooks and must be used for verification.

___


## Quick Reference

### Environment Variables

```bash
# Backend (NEVER expose these)
PAYSTACK_SECRET_KEY=sk_test_xxx   # or sk_live_xxx for production
PAYSTACK_WEBHOOK_SECRET=whsec_xxx # Optional: separate webhook secret

# Frontend (safe to expose)
PAYSTACK_PUBLIC_KEY=pk_test_xxx   # or pk_live_xxx for production
```

### API Configuration

```typescript
const PAYSTACK_BASE_URL = 'https://api.paystack.co';

const headers = {
  Authorization: `Bearer ${process.env.PAYSTACK_SECRET_KEY}`,
  'Content-Type': 'application/json',
};
```

### Reference Management (CRITICAL)

```typescript
// ❌ WRONG: Generate reference locally, save to DB, then call Paystack
const reference = generateReference('PAY');
await db.orders.create({ reference, ... });  // Orphan if next line fails!
const transaction = await initializeTransaction({ ..., reference });

// ✅ CORRECT: Call Paystack first, then save with returned reference
const transaction = await initializeTransaction({
  email,
  amount: amountInKobo,
  // reference intentionally omitted — Paystack generates it
});
await db.orders.create({
  reference: transaction.reference,  // ← Use Paystack's reference
  ...
});
```

**Flow:**
1. Call `initializeTransaction()` WITHOUT a reference parameter
2. Paystack returns a unique reference in `result.data.reference`
3. Store this reference in your database
4. Return this reference to frontend
5. Webhook events will contain this same reference

⚠️ **WHY**: If you save to DB before calling Paystack and the API fails, you create orphan records. By calling Paystack first, you only persist successful initializations.

### Currency: Kobo Conversion (CRITICAL)

```typescript
// Paystack amounts are in smallest currency unit
// NGN: kobo (1 Naira = 100 kobo)
// GHS: pesewas (1 Cedi = 100 pesewas)

const amountInKobo = amountInNaira * 100;  // 5000 NGN = 500000 kobo
const amountInNaira = amountInKobo / 100;  // 500000 kobo = 5000 NGN

// Helper function
function toKobo(amount: number): number {
  return Math.round(amount * 100);
}
```

### Supported Payment Channels

| Currency | Smallest Unit | Multiplier |
|----------|---------------|------------|
| NGN      | kobo          | 100        |
| GHS      | pesewas       | 100        |
| ZAR      | cents         | 100        |
| KES      | cents         | 100        |
| USD      | cents         | 100        |

Note: multiplier is 100 for all listed currencies; store and compare amounts in the smallest unit.

```typescript
type PaystackChannel = 'card' | 'bank' | 'ussd' | 'qr' | 'mobile_money' | 'bank_transfer' | 'eft';
```

---

## Payment Flow Overview

### One-Time Payment Flow

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Client    │────▶│   Backend   │────▶│  Paystack   │────▶│   Backend   │
│  (initiate) │     │ (initialize)│     │  (payment)  │     │  (verify)   │
└─────────────┘     └─────────────┘     └─────────────┘     └─────────────┘
      │                    │                   │                   │
      │  1. Request        │  2. POST          │  3. Pay via       │  4. GET
      │     payment        │     /initialize   │     popup/redirect│     /verify/:ref
      │                    │                   │                   │
      │                    │  Returns:         │  Returns:         │  Returns:
      │                    │  authorization_url│  reference        │  status: success
      │                    │  access_code      │                   │  amount verified
      │                    │  reference        │                   │
```

### Decision: Redirect vs Popup

| Method | Use When | Pros | Cons |
|--------|----------|------|------|
| **Redirect** | Simple integration, server-rendered apps | No JS required, works everywhere | User leaves your site |
| **Popup** | SPA, better UX | User stays on your site, smoother | Requires Inline.js SDK |

### Subscription Flow

```
1. Create Plan (Dashboard or API) → plan_code
2. Create/Fetch Customer → customer_code
3. Initialize with plan parameter OR Create subscription directly
4. Handle webhook events for lifecycle management
```

See [references/one-time-payments.md](references/one-time-payments.md) for detailed one-time payment implementation.
See [references/subscriptions.md](references/subscriptions.md) for subscription/recurring billing.

---

## Core Implementation

### TypeScript Interfaces

```typescript
interface PaystackResponse<T> {
  status: boolean;
  message: string;
  data: T;
}

interface InitializeTransactionData {
  authorization_url: string;
  access_code: string;
  reference: string;
}

interface VerifyTransactionData {
  id: number;
  status: 'success' | 'failed' | 'abandoned';
  reference: string;
  amount: number; // in kobo
  currency: string;
  channel: string;
  customer: {
    id: number;
    email: string;
    customer_code: string;
  };
  authorization?: {
    authorization_code: string;
    card_type: string;
    last4: string;
    exp_month: string;
    exp_year: string;
    reusable: boolean;
  };
  metadata?: Record<string, unknown>;
}

interface WebhookEvent {
  event: string;
  data: Record<string, unknown>;
}
```

### Initialize Transaction

```typescript
interface InitializeParams {
  email: string;
  amount: number; // in kobo
  /**
   * Transaction reference. RECOMMENDED: Omit this parameter.
   * When omitted, Paystack generates a unique reference.
   * Use the returned reference for database storage.
   * Only provide a reference for retry scenarios (e.g., retrying after network timeout).
   */
  reference?: string;
  callback_url?: string;
  metadata?: Record<string, unknown>;
  channels?: PaystackChannel[];
  plan?: string; // plan_code for subscriptions
}

/**
 * Initialize a Paystack transaction.
 *
 * IMPORTANT: Omit the 'reference' parameter to let Paystack generate one.
 * Store the returned reference (result.data.reference) in your database.
 * This reference will appear in webhooks and verification responses.
 */
async function initializeTransaction(params: InitializeParams): Promise<InitializeTransactionData> {
  const response = await fetch(`${PAYSTACK_BASE_URL}/transaction/initialize`, {
    method: 'POST',
    headers,
    body: JSON.stringify(params),
  });

  const result: PaystackResponse<InitializeTransactionData> = await response.json();

  if (!result.status) {
    throw new Error(result.message);
  }

  return result.data;
}
```

### Verify Transaction (CRITICAL: Always Verify Amount)

```typescript
async function verifyTransaction(reference: string, expectedAmount: number): Promise<VerifyTransactionData> {
  const response = await fetch(
    `${PAYSTACK_BASE_URL}/transaction/verify/${encodeURIComponent(reference)}`,
    { headers }
  );

  const result: PaystackResponse<VerifyTransactionData> = await response.json();

  if (!result.status) {
    throw new Error(result.message);
  }

  // CRITICAL: Verify payment status
  if (result.data.status !== 'success') {
    throw new Error(`Payment not successful: ${result.data.status}`);
  }

  // CRITICAL: Verify amount matches expected (prevents underpayment fraud)
  if (result.data.amount !== expectedAmount) {
    throw new Error(`Amount mismatch: expected ${expectedAmount}, got ${result.data.amount}`);
  }

  return result.data;
}
```

---

## Webhook Essentials

Webhooks notify your server of payment events. **Signature verification is mandatory for security.**

### Signature Verification (SECURITY CRITICAL)

```typescript
import crypto from 'crypto';

function verifyPaystackWebhook(
  payload: string | Buffer,
  signature: string,
  secretKey: string
): boolean {
  const payloadString = typeof payload === 'string' ? payload : payload.toString();

  const hash = crypto
    .createHmac('sha512', secretKey)
    .update(payloadString)
    .digest('hex');

  // Use timing-safe comparison to prevent timing attacks
  try {
    return crypto.timingSafeEqual(
      Buffer.from(hash, 'hex'),
      Buffer.from(signature.toLowerCase(), 'hex')
    );
  } catch {
    return false; // Lengths don't match
  }
}

// Usage in webhook handler
function handleWebhook(req: Request): Response {
  const signature = req.headers.get('x-paystack-signature');

  if (!signature) {
    return new Response('Missing signature', { status: 401 });
  }

  // IMPORTANT: Use raw body, not parsed JSON
  const rawBody = await req.text();

  const isValid = verifyPaystackWebhook(
    rawBody,
    signature,
    process.env.PAYSTACK_SECRET_KEY!
  );

  if (!isValid) {
    console.error('Invalid webhook signature');
    return new Response('Invalid signature', { status: 401 });
  }

  const event: WebhookEvent = JSON.parse(rawBody);

  // Process event (see below)
  await processWebhookEvent(event);

  // MUST return 200 within 30 seconds
  return new Response('OK', { status: 200 });
}
```

### Key Webhook Events

| Event | When | Action |
|-------|------|--------|
| `charge.success` | Payment completed | Fulfill order, update database |
| `charge.failed` | Payment failed | Notify user, log for analysis |
| `subscription.create` | New subscription | Activate subscription |
| `subscription.disable` | Subscription cancelled | Revoke access |
| `subscription.not_renew` | Won't renew (user cancelled) | Send retention email |
| `invoice.create` | 3 days before charge | Send reminder email |
| `invoice.payment_failed` | Subscription charge failed | Notify user, retry logic |
| `invoice.update` | Invoice status changed | Update subscription status |

### Idempotency

```typescript
async function processWebhookEvent(event: WebhookEvent): Promise<void> {
  // Extract unique event identifier
  const eventId = event.data.id as number;

  // Check if already processed (use your database)
  const processed = await db.webhookEvents.findUnique({ where: { eventId } });
  if (processed) {
    console.log(`Event ${eventId} already processed, skipping`);
    return;
  }

  // Process based on event type
  switch (event.event) {
    case 'charge.success':
      await handleChargeSuccess(event.data);
      break;
    case 'subscription.create':
      await handleSubscriptionCreate(event.data);
      break;
    // ... other events
  }

  // Mark as processed
  await db.webhookEvents.create({ data: { eventId, event: event.event } });
}
```

See [references/webhooks.md](references/webhooks.md) for complete event handling patterns.

---

## Framework Guides

### Next.js (App Router)

Recommended file structure:
```
app/api/paystack/
├── initialize/route.ts    # POST - Initialize payment
├── verify/route.ts        # GET - Verify payment
└── webhook/route.ts       # POST - Handle webhooks
components/
└── PaystackButton.tsx     # Client component with Inline.js
lib/
└── paystack.ts            # Utility functions
```

See [references/nextjs-implementation.md](references/nextjs-implementation.md) for complete implementation.

### Express.js

Recommended structure:
```
routes/
└── paystack.routes.ts
controllers/
└── paystack.controller.ts
middleware/
└── paystack.middleware.ts  # Signature verification
services/
└── paystack.service.ts     # API calls
```

See [references/express-implementation.md](references/express-implementation.md) for complete implementation.

---

## Deployment Checklist

### Before Going Live

- [ ] **Environment Variables**
  - [ ] `PAYSTACK_SECRET_KEY` set (sk_live_xxx for production)
  - [ ] `PAYSTACK_PUBLIC_KEY` set (pk_live_xxx for production)
  - [ ] Keys not committed to version control

- [ ] **Webhook Configuration**
  - [ ] Webhook URL configured in Paystack Dashboard (Settings → API Keys & Webhooks)
  - [ ] HTTPS endpoint (required for production)
  - [ ] Signature verification implemented and tested
  - [ ] Return 200 OK within 30 seconds

- [ ] **Payment Verification**
  - [ ] Amount verification implemented (prevent underpayment)
  - [ ] Status check implemented (only process `success`)
  - [ ] Idempotency for webhook handling

- [ ] **Testing**
  - [ ] Test with Paystack test cards
  - [ ] Test webhook with ngrok (local) or staging URL
  - [ ] Test subscription lifecycle (create → charge → cancel)
  - [ ] Test failed payment scenarios

- [ ] **Monitoring**
  - [ ] Webhook delivery logs in Paystack Dashboard
  - [ ] Application logging for payment events
  - [ ] Alerting for webhook failures

### Test Cards

| Card Number | Scenario |
|-------------|----------|
| `4084084084084081` | Successful payment |
| `5078575078575078` | Insufficient funds |
| `4084080000005408` | Declined |

CVV: Any 3 digits | Expiry: Any future date | PIN: 1234 | OTP: 123456

---

## Quick Troubleshooting

### Common Errors

| Error | Cause | Solution |
|-------|-------|----------|
| `Invalid key` | Wrong API key | Check env var, test vs live key |
| `Amount is required` | Missing/zero amount | Ensure amount in kobo (multiply by 100) |
| `Invalid signature` | Webhook verification failed | Use raw body, verify secret key |
| `Transaction reference exists` | Duplicate reference | Generate unique reference per transaction |
| `Customer not found` | Invalid customer_code | Create customer first or use email |

### Debug Checklist

1. **API calls failing?**
   - Check Authorization header format: `Bearer sk_xxx`
   - Verify you're using the correct base URL
   - Check Paystack status page for outages

2. **Webhook not received?**
   - Verify URL is publicly accessible (HTTPS)
   - Check Paystack Dashboard → Webhooks for delivery logs
   - Ensure you return 200 OK quickly

3. **Amount issues?**
   - Remember: amounts in kobo (multiply by 100)
   - Verify amount in callback matches expected

See [references/troubleshooting.md](references/troubleshooting.md) for comprehensive debugging guide.

---

## Official Documentation

- [Paystack API Reference](https://paystack.com/docs/api/)
- [Accept Payments Guide](https://paystack.com/docs/payments/accept-payments/)
- [Webhooks Guide](https://paystack.com/docs/payments/webhooks/)
- [Inline.js (Popup)](https://paystack.com/docs/developer-tools/inlinejs/)
- [Subscriptions](https://paystack.com/docs/payments/subscriptions/)
- [Test Payments](https://paystack.com/docs/payments/test-payments/)
