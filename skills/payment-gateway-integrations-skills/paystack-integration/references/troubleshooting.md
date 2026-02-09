> **When to read:** When debugging integration issues, signature failures, or payment anomalies.
> **What problem it solves:** Presents common errors, causes, and quick fixes for production issues.
> **When to skip:** Not needed during greenfield implementation unless troubleshooting.
> **Prerequisites:** Read AGENT_EXECUTION_SPEC.md first.

# Troubleshooting - Common Issues & Solutions

This guide covers common Paystack integration issues, debugging techniques, and testing strategies.

## Table of Contents

1. [Common Errors](#common-errors)
2. [API Issues](#api-issues)
3. [Webhook Issues](#webhook-issues)
4. [Payment Issues](#payment-issues)
5. [Subscription Issues](#subscription-issues)
6. [Testing](#testing)
7. [Advanced Features](#advanced-features)

---

## Common Errors

### Error Code Reference

| Error | Cause | Solution |
|-------|-------|----------|
| `Invalid key` | Wrong or missing API key | Check `PAYSTACK_SECRET_KEY` env var; ensure using correct test/live key |
| `Amount is required` | Missing amount field | Ensure amount is provided and > 0 |
| `Invalid amount` | Amount too low or not a number | Minimum is 100 kobo (₦1); ensure integer value |
| `Email is required` | Missing email field | Provide valid customer email |
| `Transaction reference exists` | Duplicate reference | Generate unique reference for each transaction |
| `Invalid signature` | Webhook verification failed | Use raw body; verify secret key matches |
| `Customer not found` | Invalid customer_code | Create customer first or use email instead |
| `Plan not found` | Invalid plan_code | Verify plan exists in Dashboard; check test vs live mode |
| `Authorization code is invalid` | Card no longer valid | Ask customer to re-authorize card |
| `Insufficient Funds` | Card declined | Customer should use different card or fund account |

### Kobo Conversion Errors

**Problem:** Payment amount is wrong (100x too small or too large)

```typescript
// WRONG - amount in Naira
const transaction = await initializeTransaction({
  email: 'user@example.com',
  amount: 5000, // This is 50 kobo, not ₦5000!
});

// CORRECT - amount in kobo
const transaction = await initializeTransaction({
  email: 'user@example.com',
  amount: 5000 * 100, // ₦5000 = 500000 kobo
});

// BEST - use helper function
function toKobo(naira: number): number {
  return Math.round(naira * 100);
}

const transaction = await initializeTransaction({
  email: 'user@example.com',
  amount: toKobo(5000), // Clearly ₦5000
});
```

### Reference Collision

**Problem:** `Transaction reference exists` error

```typescript
// WRONG - static or predictable reference
const ref = 'order_123'; // Will fail on retry

// CORRECT - unique reference per transaction
function generateReference(): string {
  const timestamp = Date.now().toString(36);
  const random = Math.random().toString(36).substring(2, 8);
  return `TXN_${timestamp}_${random}`.toUpperCase();
}

// Or use crypto
import crypto from 'crypto';
const ref = `TXN_${crypto.randomBytes(8).toString('hex')}`;
```

---

## API Issues

### Authentication Errors

**Problem:** `Invalid key` or 401 Unauthorized

```typescript
// Check 1: Correct header format
const headers = {
  Authorization: `Bearer ${process.env.PAYSTACK_SECRET_KEY}`, // Note: Bearer with space
  'Content-Type': 'application/json',
};

// Check 2: Key starts with sk_ (not pk_)
console.log('Key prefix:', process.env.PAYSTACK_SECRET_KEY?.substring(0, 8));
// Should be: sk_test_ or sk_live_

// Check 3: Environment variable is set
if (!process.env.PAYSTACK_SECRET_KEY) {
  throw new Error('PAYSTACK_SECRET_KEY not set');
}

// Check 4: No extra whitespace
const key = process.env.PAYSTACK_SECRET_KEY.trim();
```

### Test vs Live Mode

**Problem:** Transactions not appearing, or test cards rejected

```typescript
// Test mode keys
PAYSTACK_SECRET_KEY=sk_test_xxx  // Starts with sk_test_
PAYSTACK_PUBLIC_KEY=pk_test_xxx  // Starts with pk_test_

// Live mode keys
PAYSTACK_SECRET_KEY=sk_live_xxx  // Starts with sk_live_
PAYSTACK_PUBLIC_KEY=pk_live_xxx  // Starts with pk_live_

// Test cards only work with test keys
// Live cards only work with live keys
```

### Network Errors

**Problem:** `ECONNREFUSED`, `ETIMEDOUT`, or network errors

```typescript
// Add timeout and retry logic
async function paystackRequest(url: string, options: RequestInit, retries = 3) {
  const controller = new AbortController();
  const timeout = setTimeout(() => controller.abort(), 30000); // 30s timeout

  try {
    const response = await fetch(url, {
      ...options,
      signal: controller.signal,
    });
    clearTimeout(timeout);
    return response;
  } catch (error) {
    clearTimeout(timeout);

    if (retries > 0 && (error as Error).name === 'AbortError') {
      console.log(`Retrying... (${retries} attempts left)`);
      await new Promise(r => setTimeout(r, 1000));
      return paystackRequest(url, options, retries - 1);
    }

    throw error;
  }
}
```

---

## Webhook Issues

### Webhook Not Received

**Checklist:**

1. **URL is publicly accessible**
   ```bash
   # Test from external network
   curl -X POST https://yourdomain.com/api/paystack/webhook \
     -H "Content-Type: application/json" \
     -d '{"test": true}'
   ```

2. **HTTPS is required** (for production)
   - Paystack only sends webhooks to HTTPS URLs
   - Use ngrok for local development

3. **Check Paystack Dashboard**
   - Go to Settings → API Keys & Webhooks
   - View webhook delivery logs
   - Check for failed deliveries and error messages

4. **Firewall/Security rules**
   - Whitelist Paystack IPs if using IP restrictions
   - Check WAF rules aren't blocking POST requests

### Signature Verification Failing

**Problem:** `Invalid signature` even with correct key

```typescript
// COMMON MISTAKE 1: Using parsed JSON instead of raw body
// WRONG
app.use(express.json());
app.post('/webhook', (req, res) => {
  const signature = req.headers['x-paystack-signature'];
  const isValid = verify(JSON.stringify(req.body), signature); // WRONG!
});

// CORRECT - Use raw body
app.post('/webhook',
  express.raw({ type: 'application/json' }),
  (req, res) => {
    const rawBody = req.body; // Buffer
    const signature = req.headers['x-paystack-signature'];
    const isValid = verify(rawBody, signature); // Correct
  }
);

// COMMON MISTAKE 2: Using public key instead of secret key
const hash = crypto
  .createHmac('sha512', process.env.PAYSTACK_SECRET_KEY) // Must be SECRET key
  .update(payload)
  .digest('hex');

// COMMON MISTAKE 3: Case sensitivity
// Compare lowercase to lowercase
return hash.toLowerCase() === signature.toLowerCase();
```

### Webhook Timeout

**Problem:** Paystack marks webhook as failed (retries)

```typescript
// WRONG - Heavy processing before response
app.post('/webhook', async (req, res) => {
  await updateDatabase();
  await sendEmails();
  await processOrder();
  res.json({ ok: true }); // Too late! >30s
});

// CORRECT - Respond immediately, process async
app.post('/webhook', async (req, res) => {
  // Respond immediately
  res.json({ received: true });

  // Process in background
  processWebhookAsync(req.body).catch(console.error);
});

// Or use a queue
app.post('/webhook', async (req, res) => {
  await queue.add('paystack-webhook', req.body);
  res.json({ received: true });
});
```

---

## Payment Issues

### Payment Page Not Loading

**Problem:** User sees error on Paystack payment page

1. **Check amount** - Must be ≥ 100 kobo
2. **Check email** - Must be valid format
3. **Check callback_url** - Must be valid URL
4. **Check channels** - Must be valid array or omit

### Payment Successful But Not Verified

**Problem:** User paid but your system shows unpaid

```typescript
// 1. Don't rely solely on callback URL
// Users can manipulate URLs or close browser before redirect

// 2. ALWAYS verify server-side
async function handleCallback(reference: string) {
  // Don't trust callback parameters
  const transaction = await verifyTransaction(reference);

  if (transaction.status === 'success') {
    // Now it's safe to fulfill
  }
}

// 3. Use webhooks as backup
// charge.success webhook will arrive even if callback fails

// 4. Check verification response thoroughly
if (transaction.status !== 'success') {
  // Payment not complete
  return;
}

if (transaction.amount !== expectedAmount) {
  // Amount mismatch - possible fraud
  return;
}
```

### Card Declined

Common decline reasons and solutions:

| Gateway Response | Meaning | Solution |
|------------------|---------|----------|
| `Insufficient Funds` | Not enough money | Use different card |
| `Declined` | Bank rejected | Contact bank or use different card |
| `Invalid Card` | Card number wrong | Re-enter card details |
| `Expired Card` | Card past expiry | Use valid card |
| `Card Not Enrolled` | 3DS not set up | Customer should enable with bank |

---

## Subscription Issues

### Subscription Not Created

**Problem:** Payment successful but no subscription

```typescript
// Ensure plan parameter is passed
const transaction = await initializeTransaction({
  email: 'user@example.com',
  amount: planAmount, // Must match plan amount exactly
  plan: 'PLN_xxx', // Plan code required
});

// Check plan exists
const plan = await getPlan('PLN_xxx');
console.log('Plan amount:', plan.amount);
console.log('Transaction amount:', transactionAmount);
// These must match!
```

### Subscription Charge Failed

**Problem:** `invoice.payment_failed` webhook received

```typescript
// Paystack does NOT automatically retry failed subscription charges
// You must handle this:

async function handleInvoicePaymentFailed(data: {
  customer: { email: string };
  subscription: { subscription_code: string };
  attempt: number;
}) {
  const { email } = data.customer;

  // 1. Notify customer
  await sendEmail({
    to: email,
    subject: 'Payment failed - Please update your card',
    template: 'payment-failed',
  });

  // 2. Provide manage link
  const link = await generateManageLink(data.subscription.subscription_code);

  // 3. Consider grace period before revoking access
  await updateUserStatus(email, 'payment_issue');

  // 4. After X failed attempts, Paystack will cancel
  // You'll receive subscription.disable webhook
}
```

### Cannot Cancel Subscription

**Problem:** `email_token` not available

```typescript
// email_token is returned when subscription is created
// Store it along with subscription_code

// If lost, fetch subscription to get it
const subscription = await getSubscription('SUB_xxx');
const emailToken = subscription.email_token;

// Or generate manage link (doesn't need token)
const link = await generateManageLink('SUB_xxx');
// Send to customer - they can cancel from there
```

---

## Testing

### Test Cards

| Card Number | CVV | Expiry | PIN | OTP | Result |
|-------------|-----|--------|-----|-----|--------|
| `4084084084084081` | Any 3 digits | Any future | 1234 | 123456 | Success |
| `5078575078575078` | Any 3 digits | Any future | 1111 | - | Insufficient Funds |
| `4084080000005408` | Any 3 digits | Any future | 1234 | 123456 | Declined |

### Testing Webhooks Locally

```bash
# 1. Start your server
npm run dev  # Running on port 3000

# 2. Install and run ngrok
ngrok http 3000
# Output: https://abc123.ngrok.io

# 3. Set webhook URL in Paystack Dashboard
# Settings → API Keys & Webhooks
# URL: https://abc123.ngrok.io/api/paystack/webhook

# 4. Make a test payment
# Webhook will be sent to your local server

# 5. View ngrok logs
# Open http://localhost:4040 for webhook inspector
```

### Manual Webhook Testing

```bash
# Generate valid signature
SECRET="sk_test_xxx"
PAYLOAD='{"event":"charge.success","data":{"id":123,"reference":"test_ref","amount":500000,"status":"success","customer":{"email":"test@example.com"}}}'
SIGNATURE=$(echo -n "$PAYLOAD" | openssl dgst -sha512 -hmac "$SECRET" | cut -d' ' -f2)

# Send to your endpoint
curl -X POST http://localhost:3000/api/paystack/webhook \
  -H "Content-Type: application/json" \
  -H "x-paystack-signature: $SIGNATURE" \
  -d "$PAYLOAD"
```

### Testing Checklist

- [ ] Initialize payment with valid data
- [ ] Initialize payment with missing email (expect error)
- [ ] Initialize payment with amount < 100 (expect error)
- [ ] Complete payment with test card
- [ ] Verify successful payment
- [ ] Verify failed payment status
- [ ] Receive and process charge.success webhook
- [ ] Verify webhook signature validation
- [ ] Reject webhook with invalid signature
- [ ] Create subscription with plan
- [ ] Receive subscription.create webhook
- [ ] Cancel subscription
- [ ] Receive subscription.disable webhook

---

## Advanced Features

### Refunds

```typescript
// Full refund
async function refundTransaction(transactionId: number) {
  const response = await fetch('https://api.paystack.co/refund', {
    method: 'POST',
    headers: getPaystackHeaders(),
    body: JSON.stringify({ transaction: transactionId }),
  });

  return response.json();
}

// Partial refund
async function partialRefund(transactionId: number, amount: number) {
  const response = await fetch('https://api.paystack.co/refund', {
    method: 'POST',
    headers: getPaystackHeaders(),
    body: JSON.stringify({
      transaction: transactionId,
      amount, // Amount to refund in kobo
    }),
  });

  return response.json();
}
```

### Virtual Accounts (Dedicated NUBAN)

```typescript
// Create dedicated virtual account for customer
async function createVirtualAccount(customerCode: string) {
  const response = await fetch(
    'https://api.paystack.co/dedicated_account',
    {
      method: 'POST',
      headers: getPaystackHeaders(),
      body: JSON.stringify({
        customer: customerCode,
        preferred_bank: 'wema-bank', // or 'titan-paystack'
      }),
    }
  );

  const result = await response.json();
  return result.data;
  // Returns: { account_number, account_name, bank }
}
```

### Transfers (Payouts)

```typescript
// Create transfer recipient
async function createRecipient(params: {
  name: string;
  account_number: string;
  bank_code: string;
}) {
  const response = await fetch(
    'https://api.paystack.co/transferrecipient',
    {
      method: 'POST',
      headers: getPaystackHeaders(),
      body: JSON.stringify({
        type: 'nuban',
        ...params,
        currency: 'NGN',
      }),
    }
  );

  return response.json();
}

// Initiate transfer
async function initiateTransfer(params: {
  amount: number;
  recipient: string; // recipient_code
  reason?: string;
}) {
  const response = await fetch('https://api.paystack.co/transfer', {
    method: 'POST',
    headers: getPaystackHeaders(),
    body: JSON.stringify({
      source: 'balance',
      ...params,
    }),
  });

  return response.json();
}
```

### Bank List

```typescript
// Get list of banks for transfers
async function getBanks(country = 'nigeria') {
  const response = await fetch(
    `https://api.paystack.co/bank?country=${country}`,
    { headers: getPaystackHeaders() }
  );

  const result = await response.json();
  return result.data;
  // Returns: [{ name, code, ... }, ...]
}

// Verify account number
async function resolveAccountNumber(accountNumber: string, bankCode: string) {
  const response = await fetch(
    `https://api.paystack.co/bank/resolve?account_number=${accountNumber}&bank_code=${bankCode}`,
    { headers: getPaystackHeaders() }
  );

  const result = await response.json();
  return result.data;
  // Returns: { account_number, account_name }
}
```

---

## Debugging Tips

### Enable Logging

```typescript
// Log all Paystack requests
async function paystackFetch(url: string, options: RequestInit) {
  console.log('Paystack Request:', {
    url,
    method: options.method,
    body: options.body ? JSON.parse(options.body as string) : undefined,
  });

  const response = await fetch(url, options);
  const data = await response.json();

  console.log('Paystack Response:', {
    status: response.status,
    data,
  });

  return data;
}
```

### Check Paystack Status

If experiencing issues, check:
- [Paystack Status Page](https://status.paystack.com/)
- [Paystack Twitter](https://twitter.com/payaborode)

### Contact Support

For unresolved issues:
- Email: support@paystack.com
- Include: Transaction reference, timestamp, error message, request/response logs
