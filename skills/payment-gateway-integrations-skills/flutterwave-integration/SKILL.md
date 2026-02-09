---
name: flutterwave-integration
description: >
  Implement Flutterwave payment processing for African markets (34+ countries). Use when integrating one-time payments (card, bank transfer, USSD, mobile money), subscriptions/recurring billing, or webhook handling. Covers Next.js and Express implementations with TypeScript. Triggers: payment integration, Flutterwave setup, accept payments Africa, NGN/GHS/KES/ZAR payments, mobile money, M-Pesa, MTN, subscription billing, payment webhooks, Flutterwave API, tx_ref, verif-hash.
---


## For Coding Agents: Start Here

Before reading any examples or framework guides, you MUST read:

**[Agent Execution Spec](references/AGENT_EXECUTION_SPEC.md)**

This file defines the payment safety contract and execution order that must be followed to prevent fraud and incorrect implementations. Start by identifying what you are trying to implement:

- If implementing frontend popup/redirect → read [references/one-time-payments.md](references/one-time-payments.md)
- If implementing backend verification → read [references/express-implementation.md](references/express-implementation.md) or [references/nextjs-implementation.md](references/nextjs-implementation.md)
- If implementing webhooks → read [references/webhooks.md](references/webhooks.md)
- If implementing subscriptions → read [references/subscriptions.md](references/subscriptions.md)
- If implementing mobile money (M-Pesa, MTN, Airtel) → read [references/mobile-money.md](references/mobile-money.md)
- If implementing bank transfers/payouts → read [references/bank-transfers.md](references/bank-transfers.md)
- If unsure about responsibilities or safety → read [references/AGENT_EXECUTION_SPEC.md](references/AGENT_EXECUTION_SPEC.md) first
- If debugging issues → read [references/troubleshooting.md](references/troubleshooting.md)
- If you need v3 vs v4 migration or enckey handling → read [references/flutterwave-specific.md](references/flutterwave-specific.md)

If you are unsure about responsibilities, safety, or execution order, read: references/AGENT_EXECUTION_SPEC.md first.

---

### Condensed Payment Safety Rules

- Generate unique `tx_ref` FIRST, store in DB with status=pending, THEN call Flutterwave.
- Always convert and store amounts in smallest currency unit (kobo/pesewas/cents).
- Never trust client-side success callbacks; always verify on backend.
- Webhook handlers must verify signature (`verif-hash` header OR HMAC-SHA256) before processing.
- Verify that the verified amount exactly matches the expected DB amount before fulfilling.
- Handle `success-pending-validation` status (wait for webhook confirmation).
- Ensure idempotency: if order.status == 'paid', exit immediately.

---

## Table of Contents (all contained in ./references/)

0. [Agent Execution Spec](references/AGENT_EXECUTION_SPEC.md) - READ FIRST
1. [Quick Reference](#quick-reference)
2. [Payment Flow Overview](#payment-flow-overview)
3. [Core Implementation](#core-implementation)
4. [Webhook Essentials](#webhook-essentials)
5. [Framework Guides](#framework-guides)
6. [Deployment Checklist](#deployment-checklist)
7. [Quick Troubleshooting](#quick-troubleshooting)
8. [Database example](references/database-example.md)
9. [Local webhook testing with ngrok](references/testing-with-ngrok.md)


---

## Quick Reference

### Environment Variables

```bash
# Backend (NEVER expose these)
FLW_SECRET_KEY=FLWSECK_TEST-xxxx   # or FLWSECK-xxxx for production
FLW_SECRET_HASH=your_webhook_hash   # Set in Dashboard → Settings → Webhooks

# Frontend (safe to expose)
FLW_PUBLIC_KEY=FLWPUBK_TEST-xxxx   # or FLWPUBK-xxxx for production

# Optional
FLW_ENCRYPTION_KEY=FLWSECK_TESTxxxx  # For direct card charges only
```

### API Configuration

```typescript
const FLW_BASE_URL = 'https://api.flutterwave.com/v3';

const headers = {
  Authorization: `Bearer ${process.env.FLW_SECRET_KEY}`,
  'Content-Type': 'application/json',
};
```

### tx_ref Generation (CRITICAL)

```typescript
import crypto from 'crypto';

// ALWAYS generate tx_ref BEFORE calling Flutterwave
function generateTxRef(prefix = 'FLW'): string {
  const timestamp = Date.now().toString(36);
  const random = crypto.randomBytes(4).toString('hex');
  return `${prefix}_${timestamp}_${random}`.toUpperCase();
}

// Example: FLW_LK5J2M8_A1B2C3D4
```

**Flow:**
1. Generate unique `tx_ref`
2. Store in database with status=pending
3. Call Flutterwave `/v3/payments` with this `tx_ref`
4. Redirect user to returned `link`
5. Webhook events will contain this same `tx_ref`

Unlike Paystack, YOU generate the reference before calling the API.

### Currency Units (CRITICAL)

| Currency | Country | Smallest Unit | Multiplier |
|----------|---------|---------------|------------|
| NGN | Nigeria | kobo | 100 |
| GHS | Ghana | pesewas | 100 |
| KES | Kenya | cents | 100 |
| ZAR | South Africa | cents | 100 |
| UGX | Uganda | cents | 100 |
| XOF | Francophone | francs | 100 |
| XAF | Central Africa | francs | 100 |
| USD | International | cents | 100 |

```typescript
function toSmallestUnit(amount: number): number {
  return Math.round(amount * 100);
}
```

### Supported Payment Channels

```typescript
type FlutterwaveChannel =
  | 'card'
  | 'banktransfer'
  | 'ussd'
  | 'credit'
  | 'mobilemoneyghana'
  | 'mobilemoneyuganda'
  | 'mobilemoneyrwanda'
  | 'mobilemoneyzambia'
  | 'mobilemoneytanzania'
  | 'mobilemoneyfranco'
  | 'mpesa'
  | 'barter';
```

---

## Payment Flow Overview

### One-Time Payment Flow

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Client    │────▶│   Backend   │────▶│ Flutterwave │────▶│   Backend   │
│  (initiate) │     │ (initialize)│     │  (payment)  │     │  (verify)   │
└─────────────┘     └─────────────┘     └─────────────┘     └─────────────┘
      │                    │                   │                   │
      │  1. Request        │  2. POST          │  3. Pay via       │  4. GET
      │     payment        │     /v3/payments  │     hosted page   │     /v3/transactions
      │                    │                   │                   │     /:id/verify
      │                    │  Returns:         │  Returns:         │
      │                    │  link (hosted)    │  tx_ref, flw_ref  │  Returns:
      │                    │                   │  transaction_id   │  status: successful
```

### Decision: Hosted vs Inline

| Method | Use When | Pros | Cons |
|--------|----------|------|------|
| **Hosted (Redirect)** | Simple integration, server-rendered | No JS required, PCI compliant | User leaves your site |
| **Inline (Popup)** | SPA, better UX | User stays on site | Requires Inline.js SDK |

---

## Core Implementation

### TypeScript Interfaces

```typescript
interface FlutterwaveResponse<T> {
  status: 'success' | 'error';
  message: string;
  data: T;
}

interface InitializePaymentData {
  link: string;  // Hosted payment page URL
}

interface VerifyTransactionData {
  id: number;
  tx_ref: string;
  flw_ref: string;
  status: 'successful' | 'pending' | 'failed' | 'success-pending-validation';
  amount: number;
  currency: string;
  charged_amount: number;
  app_fee: number;
  customer: {
    id: number;
    email: string;
    name: string;
    phone_number: string;
  };
  card?: {
    first_6digits: string;
    last_4digits: string;
    type: string;
    expiry: string;
  };
  meta?: Record<string, unknown>;
  created_at: string;
}

interface WebhookEvent {
  event: string;
  data: VerifyTransactionData;
}
```

### Initialize Transaction

```typescript
interface InitializeParams {
  tx_ref: string;           // YOUR unique reference (required)
  amount: number;           // In smallest unit
  currency: string;         // NGN, GHS, KES, etc.
  redirect_url: string;     // Where to redirect after payment
  customer: {
    email: string;
    name?: string;
    phonenumber?: string;
  };
  customizations?: {
    title?: string;
    description?: string;
    logo?: string;
  };
  payment_options?: string; // Comma-separated: "card,banktransfer,ussd"
  meta?: Record<string, unknown>;
}

async function initializePayment(params: InitializeParams): Promise<string> {
  const response = await fetch(`${FLW_BASE_URL}/payments`, {
    method: 'POST',
    headers,
    body: JSON.stringify(params),
  });

  const result: FlutterwaveResponse<InitializePaymentData> = await response.json();

  if (result.status !== 'success') {
    throw new Error(result.message);
  }

  return result.data.link;  // Redirect URL
}
```

### Verify Transaction (CRITICAL)

```typescript
async function verifyTransaction(
  transactionId: number,
  expectedAmount: number,
  expectedCurrency: string
): Promise<VerifyTransactionData> {
  const response = await fetch(
    `${FLW_BASE_URL}/transactions/${transactionId}/verify`,
    { headers }
  );

  const result: FlutterwaveResponse<VerifyTransactionData> = await response.json();

  if (result.status !== 'success') {
    throw new Error(result.message);
  }

  // CRITICAL: Verify payment status
  if (result.data.status !== 'successful') {
    throw new Error(`Payment not successful: ${result.data.status}`);
  }

  // CRITICAL: Verify amount matches expected
  if (result.data.amount !== expectedAmount) {
    throw new Error(`Amount mismatch: expected ${expectedAmount}, got ${result.data.amount}`);
  }

  // CRITICAL: Verify currency matches
  if (result.data.currency !== expectedCurrency) {
    throw new Error(`Currency mismatch: expected ${expectedCurrency}, got ${result.data.currency}`);
  }

  return result.data;
}
```

---

## Webhook Essentials

Webhooks notify your server of payment events. **Signature verification is mandatory for security.**

### Webhook Verification (Two Methods)

**Method 1: Simple verif-hash (Recommended)**

```typescript
function handleWebhook(req: Request): Response {
  const signature = req.headers.get('verif-hash');
  const secretHash = process.env.FLW_SECRET_HASH;

  if (!signature || signature !== secretHash) {
    return new Response('Invalid signature', { status: 401 });
  }

  const event: WebhookEvent = await req.json();
  await processWebhookEvent(event);
  return new Response('OK', { status: 200 });
}
```

**Method 2: HMAC-SHA256 (More Secure)**

```typescript
import crypto from 'crypto';

function verifyFlutterwaveSignature(
  payload: string,
  signature: string,
  secretHash: string
): boolean {
  const hash = crypto
    .createHmac('sha256', secretHash)
    .update(payload)
    .digest('base64');

  // Use timing-safe comparison
  try {
    return crypto.timingSafeEqual(
      Buffer.from(hash),
      Buffer.from(signature)
    );
  } catch {
    return false;
  }
}
```

### Key Webhook Events

| Event | When | Action |
|-------|------|--------|
| `charge.completed` | Payment completed | Fulfill order, update database |
| `charge.failed` | Payment failed | Notify user, log for analysis |
| `transfer.completed` | Payout completed | Update transfer status |
| `transfer.failed` | Payout failed | Retry or notify admin |
| `subscription.cancelled` | Subscription cancelled | Revoke access |

### Idempotency

```typescript
async function processWebhookEvent(event: WebhookEvent): Promise<void> {
  const txRef = event.data.tx_ref;

  // Check if already processed
  const order = await db.orders.findUnique({ where: { txRef } });

  if (!order) {
    console.log(`Order not found for tx_ref: ${txRef}`);
    return;
  }

  if (order.status === 'paid') {
    console.log(`Order ${txRef} already paid, skipping`);
    return;
  }

  // Process based on event type
  if (event.event === 'charge.completed' && event.data.status === 'successful') {
    await fulfillOrder(order, event.data);
  }
}
```

See [references/webhooks.md](references/webhooks.md) for complete event handling patterns.

---

## Framework Guides

### Next.js (App Router)

Recommended file structure:
```
app/api/flutterwave/
├── initialize/route.ts    # POST - Initialize payment
├── verify/route.ts        # GET - Verify payment
└── webhook/route.ts       # POST - Handle webhooks
components/
└── FlutterwaveButton.tsx  # Client component
lib/
└── flutterwave.ts         # Utility functions
```

See [references/nextjs-implementation.md](references/nextjs-implementation.md) for complete implementation.

### Express.js

Recommended structure:
```
routes/
└── flutterwave.routes.ts
controllers/
└── flutterwave.controller.ts
middleware/
└── flutterwave.middleware.ts  # Signature verification
services/
└── flutterwave.service.ts     # API calls
```

See [references/express-implementation.md](references/express-implementation.md) for complete implementation.

---

## Deployment Checklist

### Before Going Live

- [ ] **Environment Variables**
  - [ ] `FLW_SECRET_KEY` set (FLWSECK-xxxx for production)
  - [ ] `FLW_PUBLIC_KEY` set (FLWPUBK-xxxx for production)
  - [ ] `FLW_SECRET_HASH` set (from Dashboard)
  - [ ] Keys not committed to version control

- [ ] **Webhook Configuration**
  - [ ] Webhook URL configured in Flutterwave Dashboard
  - [ ] HTTPS endpoint (required for production)
  - [ ] Signature verification implemented
  - [ ] Return 200 OK within 30 seconds

- [ ] **Payment Verification**
  - [ ] Amount verification implemented
  - [ ] Currency verification implemented
  - [ ] Status check implemented (only process `successful`)
  - [ ] Handle `success-pending-validation` status
  - [ ] Idempotency for webhook handling

- [ ] **Testing**
  - [ ] Test with Flutterwave test cards
  - [ ] Test mobile money flows
  - [ ] Test webhook with ngrok (local)
  - [ ] Test failed payment scenarios

### Test Cards

| Card Number | CVV | Expiry | PIN | OTP | Result |
|-------------|-----|--------|-----|-----|--------|
| `5531886652142950` | 564 | 09/32 | 3310 | 12345 | Success |
| `5258585922666506` | 883 | 09/31 | 3310 | 12345 | Insufficient Funds |
| `5399838383838381` | 470 | 10/31 | 3310 | 12345 | Declined |

### Test Mobile Money

- Phone: `0551234987` (Ghana)
- OTP: `123456`

---

## Quick Troubleshooting

### Common Errors

| Error | Cause | Solution |
|-------|-------|----------|
| `Invalid API key` | Wrong or missing key | Check FLW_SECRET_KEY env var |
| `Invalid tx_ref` | Duplicate or malformed | Generate unique tx_ref per transaction |
| `Amount mismatch` | Amount verification failed | Store and compare in smallest unit |
| `Invalid signature` | Webhook verification failed | Check FLW_SECRET_HASH matches dashboard |
| `Transaction not found` | Wrong transaction ID | Use correct id from callback/webhook |

### Debug Checklist

1. **API calls failing?**
   - Check Authorization header format: `Bearer FLWSECK-xxx`
   - Verify base URL: `https://api.flutterwave.com/v3`
   - Check test vs live keys match environment

2. **Webhook not received?**
   - Verify URL is publicly accessible (HTTPS)
   - Check Flutterwave Dashboard for delivery logs
   - Ensure you return 200 OK quickly

3. **Status issues?**
   - Handle `success-pending-validation` status
   - Poll for status or wait for webhook

See [references/troubleshooting.md](references/troubleshooting.md) for comprehensive debugging guide.

---

## Official Documentation

- [Flutterwave API Reference](https://developer.flutterwave.com/)
- [Standard Payments](https://developer.flutterwave.com/docs/collecting-payments/standard)
- [Webhooks Guide](https://developer.flutterwave.com/docs/webhooks)
- [Mobile Money](https://developer.flutterwave.com/docs/mobile-money)
- [Bank Transfers](https://developer.flutterwave.com/docs/bank-transfer)
- [Test Cards](https://developer.flutterwave.com/docs/integration-guides/testing-helpers/test-cards)
