> **When to read:** When implementing webhook receivers, signature verification, and idempotent handlers.
> **What problem it solves:** Shows raw-body verification, idempotency, and event routing best practices.
> **When to skip:** If you only implement frontend initialization or client-side UI.
> **Prerequisites:** Read AGENT_EXECUTION_SPEC.md first.

# Paystack Webhooks - Complete Guide

Webhooks are HTTP callbacks that Paystack sends to your server when events occur. Proper webhook handling is **critical for security and reliability**.

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
- Manipulate subscription statuses

**Always verify the `x-paystack-signature` header.**

### Complete Verification Implementation

```typescript
import crypto from 'crypto';

/**
 * Verify Paystack webhook signature
 * @param payload - Raw request body (string or Buffer, NOT parsed JSON)
 * @param signature - Value from x-paystack-signature header
 * @param secretKey - Your PAYSTACK_SECRET_KEY
 * @returns boolean - true if signature is valid
 */
function verifyPaystackSignature(
  payload: string | Buffer,
  signature: string,
  secretKey: string
): boolean {
  if (!payload || !signature || !secretKey) {
    return false;
  }

  const payloadString = typeof payload === 'string'
    ? payload
    : payload.toString('utf8');

  const expectedSignature = crypto
    .createHmac('sha512', secretKey)
    .update(payloadString)
    .digest('hex');

  // Use timing-safe comparison to prevent timing attacks
  try {
    return crypto.timingSafeEqual(
      Buffer.from(expectedSignature, 'hex'),
      Buffer.from(signature, 'hex')
    );
  } catch {
    // Lengths don't match
    return false;
  }
}
```

### Common Signature Verification Errors

| Issue | Cause | Solution |
|-------|-------|----------|
| Signature always invalid | Using parsed JSON instead of raw body | Access raw body before any JSON parsing |
| Signature mismatch | Wrong secret key | Verify using `PAYSTACK_SECRET_KEY`, not public key |
| Encoding issues | Body encoding changed | Ensure UTF-8 encoding throughout |

### Raw Body Access Patterns

**Next.js App Router:**
```typescript
export async function POST(request: Request) {
  const rawBody = await request.text();
  const signature = request.headers.get('x-paystack-signature') || '';
  // Verify then parse
  const event = JSON.parse(rawBody);
}
```

**Express.js:**
```typescript
// Must be BEFORE bodyParser.json()
app.post('/webhook',
  express.raw({ type: 'application/json' }),
  (req, res) => {
    const rawBody = req.body; // Buffer
    const signature = req.headers['x-paystack-signature'] as string;
  }
);
```

**Node.js HTTP:**
```typescript
const chunks: Buffer[] = [];
req.on('data', chunk => chunks.push(chunk));
req.on('end', () => {
  const rawBody = Buffer.concat(chunks).toString('utf8');
  const signature = req.headers['x-paystack-signature'] as string;
});
```

---

## Event Types

### Payment Events

#### `charge.success`
Triggered when a payment is successfully completed.

```typescript
interface ChargeSuccessData {
  id: number;
  domain: 'test' | 'live';
  status: 'success';
  reference: string;
  amount: number; // in kobo
  currency: string;
  channel: string;
  gateway_response: string;
  ip_address: string;
  fees: number;
  customer: {
    id: number;
    email: string;
    customer_code: string;
    first_name: string | null;
    last_name: string | null;
  };
  authorization: {
    authorization_code: string;
    bin: string;
    last4: string;
    exp_month: string;
    exp_year: string;
    card_type: string;
    bank: string;
    country_code: string;
    brand: string;
    reusable: boolean;
    signature: string;
  };
  plan?: {
    id: number;
    name: string;
    plan_code: string;
    amount: number;
    interval: string;
  };
  metadata?: Record<string, unknown>;
  paid_at: string;
  created_at: string;
}
```

#### `charge.failed`
Triggered when a payment fails.

```typescript
interface ChargeFailedData {
  id: number;
  status: 'failed';
  reference: string;
  amount: number;
  currency: string;
  gateway_response: string; // Reason for failure
  customer: {
    id: number;
    email: string;
    customer_code: string;
  };
}
```

### Subscription Events

#### `subscription.create`
Triggered when a new subscription is created (first successful payment with a plan).

```typescript
interface SubscriptionCreateData {
  id: number;
  domain: 'test' | 'live';
  status: 'active';
  subscription_code: string;
  email_token: string;
  amount: number;
  cron_expression: string;
  next_payment_date: string;
  plan: {
    id: number;
    name: string;
    plan_code: string;
    amount: number;
    interval: string;
  };
  customer: {
    id: number;
    email: string;
    customer_code: string;
  };
  authorization: {
    authorization_code: string;
    last4: string;
    card_type: string;
    bank: string;
  };
  created_at: string;
}
```

#### `subscription.disable`
Triggered when a subscription is disabled (cancelled).

```typescript
interface SubscriptionDisableData {
  id: number;
  status: 'cancelled';
  subscription_code: string;
  email_token: string;
  customer: {
    id: number;
    email: string;
    customer_code: string;
  };
}
```

#### `subscription.not_renew`
Triggered when a user opts out of subscription renewal (before cancellation).

```typescript
interface SubscriptionNotRenewData {
  id: number;
  subscription_code: string;
  email_token: string;
  customer: {
    id: number;
    email: string;
  };
}
```

### Invoice Events

#### `invoice.create`
Triggered 3 days before a subscription charge attempt.

```typescript
interface InvoiceCreateData {
  id: number;
  domain: 'test' | 'live';
  status: 'pending';
  subscription: {
    id: number;
    subscription_code: string;
    status: string;
  };
  customer: {
    id: number;
    email: string;
    customer_code: string;
  };
  amount: number;
  description: string;
  due_date: string;
  created_at: string;
}
```

#### `invoice.payment_failed`
Triggered when a subscription charge fails.

```typescript
interface InvoicePaymentFailedData {
  id: number;
  status: 'failed';
  subscription: {
    id: number;
    subscription_code: string;
    status: string;
  };
  customer: {
    id: number;
    email: string;
  };
  amount: number;
  attempt: number; // Which retry attempt
  next_notification: string | null;
}
```

#### `invoice.update`
Triggered when invoice status changes (success/failed after retry).

```typescript
interface InvoiceUpdateData {
  id: number;
  status: 'success' | 'failed';
  paid: boolean;
  paid_at: string | null;
  subscription: {
    id: number;
    subscription_code: string;
    status: string;
  };
}
```

### Transfer Events (Payouts)

#### `transfer.success`
```typescript
interface TransferSuccessData {
  id: number;
  status: 'success';
  reference: string;
  amount: number;
  recipient: {
    id: number;
    name: string;
    account_number: string;
    bank_code: string;
  };
}
```

#### `transfer.failed`
```typescript
interface TransferFailedData {
  id: number;
  status: 'failed';
  reference: string;
  amount: number;
  reason: string;
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
  isProcessed(eventId: number): Promise<boolean>;
  markProcessed(eventId: number, eventType: string): Promise<void>;
}

class PaystackWebhookHandler {
  constructor(
    private secretKey: string,
    private webhookStore: WebhookStore
  ) {}

  async handleRequest(rawBody: string, signature: string): Promise<{ status: number; message: string }> {
    // 1. Verify signature
    if (!this.verifySignature(rawBody, signature)) {
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

    // 3. Check idempotency
    const eventId = event.data.id as number;
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
      // Return 500 only if you want Paystack to retry
      return { status: 200, message: 'Processed with errors' };
    }
  }

  private verifySignature(payload: string, signature: string): boolean {
    const hash = crypto
      .createHmac('sha512', this.secretKey)
      .update(payload)
      .digest('hex');
    return hash.toLowerCase() === signature.toLowerCase();
  }

  private async processEvent(event: WebhookEvent): Promise<void> {
    console.log(`Processing event: ${event.event}`);

    switch (event.event) {
      case 'charge.success':
        await this.handleChargeSuccess(event.data);
        break;
      case 'charge.failed':
        await this.handleChargeFailed(event.data);
        break;
      case 'subscription.create':
        await this.handleSubscriptionCreate(event.data);
        break;
      case 'subscription.disable':
        await this.handleSubscriptionDisable(event.data);
        break;
      case 'subscription.not_renew':
        await this.handleSubscriptionNotRenew(event.data);
        break;
      case 'invoice.create':
        await this.handleInvoiceCreate(event.data);
        break;
      case 'invoice.payment_failed':
        await this.handleInvoicePaymentFailed(event.data);
        break;
      case 'invoice.update':
        await this.handleInvoiceUpdate(event.data);
        break;
      case 'transfer.success':
        await this.handleTransferSuccess(event.data);
        break;
      case 'transfer.failed':
        await this.handleTransferFailed(event.data);
        break;
      default:
        console.log(`Unhandled event type: ${event.event}`);
    }
  }

  // Implement these based on your business logic
  private async handleChargeSuccess(data: Record<string, unknown>): Promise<void> {
    const reference = data.reference as string;
    const amount = data.amount as number;
    const customerEmail = (data.customer as { email: string }).email;

    // Example: Update order status
    // await db.orders.update({
    //   where: { reference },
    //   data: { status: 'paid', paidAt: new Date() }
    // });

    // Example: Send confirmation email
    // await emailService.sendPaymentConfirmation(customerEmail, amount / 100);

    console.log(`Payment successful: ${reference} for ${amount / 100} NGN`);
  }

  private async handleChargeFailed(data: Record<string, unknown>): Promise<void> {
    const reference = data.reference as string;
    const reason = data.gateway_response as string;

    // Update order status, notify user, log for analysis
    console.log(`Payment failed: ${reference} - ${reason}`);
  }

  private async handleSubscriptionCreate(data: Record<string, unknown>): Promise<void> {
    const subscriptionCode = data.subscription_code as string;
    const customerEmail = (data.customer as { email: string }).email;
    const planCode = (data.plan as { plan_code: string }).plan_code;

    // Activate user subscription in your system
    console.log(`Subscription created: ${subscriptionCode} for ${customerEmail}`);
  }

  private async handleSubscriptionDisable(data: Record<string, unknown>): Promise<void> {
    const subscriptionCode = data.subscription_code as string;

    // Revoke user access, update subscription status
    console.log(`Subscription cancelled: ${subscriptionCode}`);
  }

  private async handleSubscriptionNotRenew(data: Record<string, unknown>): Promise<void> {
    const subscriptionCode = data.subscription_code as string;
    const customerEmail = (data.customer as { email: string }).email;

    // User opted out - send retention email
    console.log(`Subscription won't renew: ${subscriptionCode}`);
  }

  private async handleInvoiceCreate(data: Record<string, unknown>): Promise<void> {
    const customerEmail = (data.customer as { email: string }).email;
    const amount = data.amount as number;
    const dueDate = data.due_date as string;

    // Send upcoming charge reminder
    console.log(`Invoice created: ${amount / 100} NGN due ${dueDate}`);
  }

  private async handleInvoicePaymentFailed(data: Record<string, unknown>): Promise<void> {
    const customerEmail = (data.customer as { email: string }).email;
    const attempt = data.attempt as number;

    // Notify user, consider grace period
    console.log(`Invoice payment failed: attempt ${attempt}`);
  }

  private async handleInvoiceUpdate(data: Record<string, unknown>): Promise<void> {
    const status = data.status as string;
    const subscriptionCode = (data.subscription as { subscription_code: string }).subscription_code;

    // Update subscription status based on final invoice result
    console.log(`Invoice updated: ${subscriptionCode} - ${status}`);
  }

  private async handleTransferSuccess(data: Record<string, unknown>): Promise<void> {
    const reference = data.reference as string;
    console.log(`Transfer successful: ${reference}`);
  }

  private async handleTransferFailed(data: Record<string, unknown>): Promise<void> {
    const reference = data.reference as string;
    const reason = data.reason as string;
    console.log(`Transfer failed: ${reference} - ${reason}`);
  }
}

export { PaystackWebhookHandler };
```

---

## Best Practices

### 1. Respond Quickly (< 30 seconds)

Paystack expects a 200 response within 30 seconds. For heavy operations:

```typescript
// Queue the work, respond immediately
async function handleWebhook(event: WebhookEvent): Promise<Response> {
  // Verify signature first

  // Queue for async processing
  await queue.add('paystack-webhook', event);

  // Respond immediately
  return new Response('OK', { status: 200 });
}

// Process in background worker
queue.process('paystack-webhook', async (job) => {
  const event = job.data;
  await processWebhookEvent(event);
});
```

### 2. Idempotency

Events may be delivered multiple times. Always check:

```typescript
// Using event ID
const eventId = event.data.id;
const exists = await db.processedEvents.findUnique({ where: { eventId } });
if (exists) return;

// Or using reference + event type
const key = `${event.data.reference}-${event.event}`;
const exists = await cache.get(key);
if (exists) return;
await cache.set(key, true, { ex: 86400 }); // 24h TTL
```

### 3. Logging

```typescript
// Log all events for debugging
console.log('Webhook received:', {
  event: event.event,
  id: event.data.id,
  reference: event.data.reference,
  timestamp: new Date().toISOString(),
});

// Log errors with context
console.error('Webhook processing failed:', {
  event: event.event,
  id: event.data.id,
  error: error.message,
  stack: error.stack,
});
```

### 4. Error Handling

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
5. Set in Paystack Dashboard: `https://abc123.ngrok.io/api/paystack/webhook`

### Manual Testing

Send test events using curl:

```bash
# Generate signature
SECRET="sk_test_xxx"
PAYLOAD='{"event":"charge.success","data":{"id":123,"reference":"test_ref","amount":50000}}'
SIGNATURE=$(echo -n "$PAYLOAD" | openssl dgst -sha512 -hmac "$SECRET" | cut -d' ' -f2)

# Send test webhook
curl -X POST http://localhost:3000/api/paystack/webhook \
  -H "Content-Type: application/json" \
  -H "x-paystack-signature: $SIGNATURE" \
  -d "$PAYLOAD"
```

### Paystack Dashboard Testing

1. Go to Paystack Dashboard → Settings → API Keys & Webhooks
2. Click "Test Webhook"
3. Select event type and send
4. View delivery status and response

### Webhook Event Logs

Monitor webhook deliveries in Paystack Dashboard:
- Settings → Webhooks → View logs
- Shows delivery status, response code, response body
- Can resend failed webhooks manually
