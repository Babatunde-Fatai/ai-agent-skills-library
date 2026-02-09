> **When to read:** When implementing webhook receivers, signature verification, and idempotent handlers.
> **What problem it solves:** Shows signature verification (both methods), idempotency, and event routing.
> **When to skip:** If you only implement frontend initialization or client-side UI.
> **Prerequisites:** Read AGENT_EXECUTION_SPEC.md first.

# Flutterwave Webhooks - Complete Guide

Webhooks are HTTP callbacks that Flutterwave sends to your server when events occur. Proper webhook handling is **critical for security and reliability**.

## Table of Contents

1. [Webhook Security](#webhook-security)
2. [Event Types](#event-types)
3. [Event Handlers](#event-handlers)
4. [Best Practices](#best-practices)
5. [Local Testing](#local-testing)

---

## Webhook Security

### Why Signature Verification Matters

Without verification, attackers can:
- Send fake payment confirmations to your webhook
- Trigger order fulfillment without actual payment
- Manipulate transaction statuses

**Always verify webhooks before processing.**

### Two Verification Methods

Flutterwave supports two methods for webhook verification: HMAC method is preferred. Use verif-hash only if raw body access is impossible.

#### Method 1: HMAC-SHA256 (flutterwave-signature header)

More secure approach using cryptographic signature verification.

```typescript
import crypto from 'crypto';

/**
 * Verify Flutterwave webhook using HMAC-SHA256
 * @param payload - Raw request body (string or Buffer, NOT parsed JSON)
 * @param signature - Value from flutterwave-signature header
 * @param secretHash - Your FLW_SECRET_HASH
 * @returns boolean - true if signature is valid
 */
function verifyFlutterwaveHMAC(
  payload: string | Buffer,
  signature: string,
  secretHash: string
): boolean {
  if (!payload || !signature || !secretHash) {
    return false;
  }

  const payloadString = typeof payload === 'string'
    ? payload
    : payload.toString('utf8');

  const expectedSignature = crypto
    .createHmac('sha256', secretHash)
    .update(payloadString)
    .digest('base64');

  // Use timing-safe comparison to prevent timing attacks
  try {
    return crypto.timingSafeEqual(
      Buffer.from(expectedSignature),
      Buffer.from(signature)
    );
  } catch {
    // Lengths don't match
    return false;
  }
}
```
#### Method 2: Simple Hash Comparison (verif-hash header)

The simplest approach - compare `verif-hash` header to your secret hash.

```typescript
function verifyWebhookSimple(
  signature: string | null,
  secretHash: string
): boolean {
  if (!signature || !secretHash) {
    return false;
  }
  return signature === secretHash;
}

// Usage in webhook handler
function handleWebhook(req: Request): Response {
  const signature = req.headers.get('verif-hash');
  const secretHash = process.env.FLW_SECRET_HASH;

  if (!verifyWebhookSimple(signature, secretHash)) {
    console.error('Invalid webhook signature');
    return new Response('Invalid signature', { status: 401 });
  }

  // Continue processing...
}
```

**Pros:** Simple, no crypto operations needed.
**Cons:** Less secure than HMAC (secret travels in plain comparison).

```

**Pros:** Cryptographically secure, resistant to timing attacks.
**Cons:** Requires raw body access before JSON parsing.

### Setting Up Secret Hash

1. Go to **Flutterwave Dashboard** → **Settings** → **Webhooks**
2. Set your Webhook URL (must be HTTPS in production)
3. Note or set your **Secret Hash** (this is your `FLW_SECRET_HASH`)

### Common Signature Verification Errors

| Issue | Cause | Solution |
|-------|-------|----------|
| Signature always invalid (HMAC) | Using parsed JSON instead of raw body | Access raw body before any JSON parsing |
| Signature mismatch | Wrong secret hash | Verify FLW_SECRET_HASH matches dashboard |
| verif-hash missing | Wrong verification method | Check if using flutterwave-signature instead |

### Raw Body Access Patterns

**Next.js App Router:**
```typescript
export async function POST(request: Request) {
  const rawBody = await request.text();
  const signature = request.headers.get('verif-hash');
  // or request.headers.get('flutterwave-signature') for HMAC

  // Verify then parse
  const event = JSON.parse(rawBody);
}
```

**Express.js:**
```typescript
// Must be BEFORE bodyParser.json() for this route
app.post('/webhook',
  express.raw({ type: 'application/json' }),
  (req, res) => {
    const rawBody = req.body; // Buffer
    const signature = req.headers['verif-hash'] as string;
  }
);
```

**Node.js HTTP:**
```typescript
const chunks: Buffer[] = [];
req.on('data', chunk => chunks.push(chunk));
req.on('end', () => {
  const rawBody = Buffer.concat(chunks).toString('utf8');
  const signature = req.headers['verif-hash'] as string;
});
```

---

## Event Types

### Payment Events

#### `charge.completed`
Triggered when a payment is successfully completed.

```typescript
interface ChargeCompletedData {
  id: number;
  tx_ref: string;
  flw_ref: string;
  device_fingerprint: string;
  amount: number;
  currency: string;
  charged_amount: number;
  app_fee: number;
  merchant_fee: number;
  processor_response: string;
  auth_model: string;
  ip: string;
  narration: string;
  status: 'successful';
  payment_type: 'card' | 'banktransfer' | 'mobilemoney' | 'ussd';
  created_at: string;
  account_id: number;
  customer: {
    id: number;
    name: string;
    phone_number: string | null;
    email: string;
    created_at: string;
  };
  card?: {
    first_6digits: string;
    last_4digits: string;
    issuer: string;
    country: string;
    type: string;
    expiry: string;
  };
  meta?: Record<string, unknown>;
}

interface ChargeCompletedEvent {
  event: 'charge.completed';
  data: ChargeCompletedData;
}
```

#### `charge.failed`
Triggered when a payment fails.

```typescript
interface ChargeFailedData {
  id: number;
  tx_ref: string;
  flw_ref: string;
  amount: number;
  currency: string;
  status: 'failed';
  processor_response: string;  // Reason for failure
  customer: {
    id: number;
    email: string;
    name: string;
  };
}
```

### Transfer Events (Payouts)

#### `transfer.completed`
Triggered when a payout transfer succeeds.

```typescript
interface TransferCompletedData {
  id: number;
  account_number: string;
  bank_name: string;
  bank_code: string;
  full_name: string;
  created_at: string;
  currency: string;
  amount: number;
  fee: number;
  status: 'SUCCESSFUL';
  reference: string;
  narration: string;
  complete_message: string;
  meta: Record<string, unknown> | null;
}
```

#### `transfer.failed`
Triggered when a payout transfer fails.

```typescript
interface TransferFailedData {
  id: number;
  account_number: string;
  bank_name: string;
  amount: number;
  currency: string;
  status: 'FAILED';
  reference: string;
  complete_message: string;  // Reason for failure
}
```

### Subscription Events

#### `subscription.cancelled`
Triggered when a subscription is cancelled.

```typescript
interface SubscriptionCancelledData {
  id: number;
  amount: number;
  customer: {
    id: number;
    customer_email: string;
  };
  plan: number;
  status: 'cancelled';
  created_at: string;
}
```

---

## Event Handlers

### Complete Webhook Handler

```typescript
import crypto from 'crypto';

interface WebhookEvent {
  event: string;
  data: Record<string, unknown>;
}

// Database interface (implement with your ORM)
interface WebhookStore {
  isProcessed(eventId: string): Promise<boolean>;
  markProcessed(eventId: string, eventType: string): Promise<void>;
}

class FlutterwaveWebhookHandler {
  constructor(
    private secretHash: string,
    private webhookStore: WebhookStore
  ) {}

  async handleRequest(
    rawBody: string,
    signature: string | null
  ): Promise<{ status: number; message: string }> {
    // 1. Verify signature
    if (!this.verifySignature(signature)) {
      console.error('Invalid webhook signature');
      return { status: 401, message: 'Invalid signature' };
    }

    // 2. Parse event
    let event: WebhookEvent;
    try {
      event = JSON.parse(rawBody);
    } catch {
      console.error('Invalid JSON payload');
      return { status: 400, message: 'Invalid JSON' };
    }

    // 3. Check idempotency using tx_ref or transfer reference
    const eventId = this.getEventId(event);
    if (await this.webhookStore.isProcessed(eventId)) {
      console.log(`Event ${eventId} already processed`);
      return { status: 200, message: 'Already processed' };
    }

    // 4. Process event
    try {
      await this.processEvent(event);
      await this.webhookStore.markProcessed(eventId, event.event);
      return { status: 200, message: 'OK' };
    } catch (error) {
      console.error('Error processing webhook:', error);
      // Return 200 to prevent retries if error is on our side
      return { status: 200, message: 'Processed with errors' };
    }
  }

  private verifySignature(signature: string | null): boolean {
    // Using simple verif-hash comparison
    return signature === this.secretHash;
  }

  private getEventId(event: WebhookEvent): string {
    // Use tx_ref for charges, reference for transfers
    return (event.data.tx_ref || event.data.reference || event.data.id) as string;
  }

  private async processEvent(event: WebhookEvent): Promise<void> {
    console.log(`Processing event: ${event.event}`);

    switch (event.event) {
      case 'charge.completed':
        await this.handleChargeCompleted(event.data);
        break;
      case 'charge.failed':
        await this.handleChargeFailed(event.data);
        break;
      case 'transfer.completed':
        await this.handleTransferCompleted(event.data);
        break;
      case 'transfer.failed':
        await this.handleTransferFailed(event.data);
        break;
      case 'subscription.cancelled':
        await this.handleSubscriptionCancelled(event.data);
        break;
      default:
        console.log(`Unhandled event type: ${event.event}`);
    }
  }

  private async handleChargeCompleted(data: Record<string, unknown>): Promise<void> {
    const txRef = data.tx_ref as string;
    const amount = data.amount as number;
    const currency = data.currency as string;
    const status = data.status as string;
    const customerEmail = (data.customer as { email: string }).email;

    // CRITICAL: Verify status is successful
    if (status !== 'successful') {
      console.log(`Charge ${txRef} status is ${status}, not processing`);
      return;
    }

    // Get order from database
    // const order = await db.orders.findUnique({ where: { txRef } });
    // if (!order) {
    //   console.error(`Order not found for tx_ref: ${txRef}`);
    //   return;
    // }

    // Check if already paid (idempotency)
    // if (order.status === 'paid') {
    //   console.log(`Order ${txRef} already paid, skipping`);
    //   return;
    // }

    // CRITICAL: Verify amount matches
    // if (order.amount !== amount || order.currency !== currency) {
    //   console.error(`Amount/currency mismatch for ${txRef}`);
    //   return;
    // }

    // Mark as paid
    // await db.orders.update({
    //   where: { txRef },
    //   data: { status: 'paid', paidAt: new Date() }
    // });

    console.log(`Payment successful: ${txRef} for ${amount} ${currency}`);
  }

  private async handleChargeFailed(data: Record<string, unknown>): Promise<void> {
    const txRef = data.tx_ref as string;
    const reason = data.processor_response as string;

    console.log(`Payment failed: ${txRef} - ${reason}`);

    // Update order status, notify user, log for analysis
  }

  private async handleTransferCompleted(data: Record<string, unknown>): Promise<void> {
    const reference = data.reference as string;
    const amount = data.amount as number;

    console.log(`Transfer successful: ${reference} for ${amount}`);

    // Update transfer status in database
  }

  private async handleTransferFailed(data: Record<string, unknown>): Promise<void> {
    const reference = data.reference as string;
    const reason = data.complete_message as string;

    console.log(`Transfer failed: ${reference} - ${reason}`);

    // Update transfer status, retry or notify admin
  }

  private async handleSubscriptionCancelled(data: Record<string, unknown>): Promise<void> {
    const customerId = (data.customer as { id: number }).id;

    console.log(`Subscription cancelled for customer ${customerId}`);

    // Revoke access, update subscription status
  }
}

export { FlutterwaveWebhookHandler };
```

---

## Best Practices

### 1. Respond Quickly (< 30 seconds)

Flutterwave expects a 200 response quickly. For heavy operations:

```typescript
// Queue the work, respond immediately
async function handleWebhook(event: WebhookEvent): Promise<Response> {
  // Verify signature first

  // Queue for async processing
  await queue.add('flutterwave-webhook', event);

  // Respond immediately
  return new Response('OK', { status: 200 });
}

// Process in background worker
queue.process('flutterwave-webhook', async (job) => {
  const event = job.data;
  await processWebhookEvent(event);
});
```

### 2. Idempotency

Events may be delivered multiple times. Always check:

```typescript
// Using tx_ref or reference
const txRef = event.data.tx_ref;

// Check if already processed
const order = await db.orders.findUnique({ where: { txRef } });
if (order?.status === 'paid') {
  console.log(`Order ${txRef} already paid, skipping`);
  return;
}

// Or use a separate processed events table
const key = `${txRef}-${event.event}`;
const exists = await db.processedEvents.findUnique({ where: { key } });
if (exists) return;
await db.processedEvents.create({ data: { key } });
```

### 3. Verify Before Fulfilling

Don't blindly trust webhook data:

```typescript
async function handleChargeCompleted(data: Record<string, unknown>) {
  const txRef = data.tx_ref as string;
  const webhookAmount = data.amount as number;

  // Fetch from YOUR database
  const order = await db.orders.findUnique({ where: { txRef } });

  if (!order) {
    throw new Error(`Order not found: ${txRef}`);
  }

  // CRITICAL: Verify amounts match
  if (order.amount !== webhookAmount) {
    throw new Error(`Amount mismatch: expected ${order.amount}, got ${webhookAmount}`);
  }

  // Now safe to fulfill
  await fulfillOrder(order);
}
```

### 4. Logging

```typescript
// Log all events for debugging
console.log('Webhook received:', {
  event: event.event,
  txRef: event.data.tx_ref,
  amount: event.data.amount,
  timestamp: new Date().toISOString(),
});

// Log errors with context
console.error('Webhook processing failed:', {
  event: event.event,
  txRef: event.data.tx_ref,
  error: error.message,
  stack: error.stack,
});
```

### 5. Error Handling

```typescript
// Return 200 for errors you can't fix with retries
// Return 500 only for transient errors you want retried

try {
  await processEvent(event);
  return new Response('OK', { status: 200 });
} catch (error) {
  if (error instanceof TransientError) {
    // Database down, network issue - retry
    return new Response('Retry', { status: 500 });
  }
  // Logic error, bad data - don't retry
  console.error('Permanent error:', error);
  return new Response('OK', { status: 200 });
}
```

---

## Local Testing

### Using ngrok

1. Install ngrok: `npm install -g ngrok`
2. Start your server: `npm run dev` (e.g., on port 3000)
3. Expose with ngrok: `ngrok http 3000`
4. Copy the HTTPS URL (e.g., `https://abc123.ngrok.io`)
5. Set in Flutterwave Dashboard: Settings → Webhooks → URL

### Manual Testing

Send test events using curl:

```bash
# Using verif-hash method
SECRET_HASH="your_secret_hash"
PAYLOAD='{"event":"charge.completed","data":{"id":123,"tx_ref":"FLW_TEST_123","flw_ref":"FLW12345","amount":500000,"currency":"NGN","status":"successful","customer":{"email":"test@example.com"}}}'

curl -X POST http://localhost:3000/api/flutterwave/webhook \
  -H "Content-Type: application/json" \
  -H "verif-hash: $SECRET_HASH" \
  -d "$PAYLOAD"
```

```bash
# Using HMAC-SHA256 method
SECRET_HASH="your_secret_hash"
PAYLOAD='{"event":"charge.completed","data":{"id":123,"tx_ref":"FLW_TEST_123","amount":500000,"status":"successful"}}'
SIGNATURE=$(echo -n "$PAYLOAD" | openssl dgst -sha256 -hmac "$SECRET_HASH" -binary | base64)

curl -X POST http://localhost:3000/api/flutterwave/webhook \
  -H "Content-Type: application/json" \
  -H "flutterwave-signature: $SIGNATURE" \
  -d "$PAYLOAD"
```

### Resend Webhook

If you missed a webhook, you can request Flutterwave to resend it:

```typescript
async function resendWebhook(transactionId: number): Promise<void> {
  const response = await fetch(
    `https://api.flutterwave.com/v3/transactions/${transactionId}/resend-hook`,
    {
      method: 'POST',
      headers: {
        Authorization: `Bearer ${process.env.FLW_SECRET_KEY}`,
      },
    }
  );

  const result = await response.json();
  console.log('Resend result:', result);
}
```

### Flutterwave Dashboard Testing

1. Go to Flutterwave Dashboard → Settings → Webhooks
2. View webhook delivery logs
3. Check delivery status and response
4. Resend failed webhooks manually
