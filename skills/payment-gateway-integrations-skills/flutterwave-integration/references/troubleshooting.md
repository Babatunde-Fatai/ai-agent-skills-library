> **When to read:** When debugging integration issues, signature failures, or payment anomalies.
> **What problem it solves:** Presents common errors, causes, and quick fixes for production issues.
> **When to skip:** Not needed during greenfield implementation unless troubleshooting.
> **Prerequisites:** Read AGENT_EXECUTION_SPEC.md first.

# Troubleshooting - Common Issues & Solutions

This guide covers common Flutterwave integration issues, debugging techniques, and testing strategies.

## Table of Contents

1. [Common Errors](#common-errors)
2. [API Issues](#api-issues)
3. [Webhook Issues](#webhook-issues)
4. [Payment Issues](#payment-issues)
5. [Testing](#testing)
6. [Debug Checklist](#debug-checklist)

---

## Common Errors

### Error Code Reference

| Error | Cause | Solution |
|-------|-------|----------|
| `Invalid API key` | Wrong or missing key | Check FLW_SECRET_KEY env var; ensure using correct test/live key |
| `Invalid tx_ref` | Duplicate or invalid reference | Generate unique tx_ref per transaction |
| `Amount is required` | Missing amount field | Ensure amount is provided and > 0 |
| `Invalid amount` | Amount too low | Check minimum amounts for currency |
| `Currency not supported` | Wrong currency code | Use supported currency (NGN, GHS, KES, etc.) |
| `Invalid signature` | Webhook verification failed | Check FLW_SECRET_HASH matches dashboard |
| `Transaction not found` | Wrong transaction ID | Use correct id from callback/webhook |
| `Insufficient funds` | Not enough balance | Check Flutterwave balance for transfers |

### tx_ref Errors

**Problem:** `tx_ref already exists` or `Invalid tx_ref`

```typescript
// WRONG - static or predictable reference
const txRef = 'order_123';  // Will fail on retry

// WRONG - reusing tx_ref
const txRef = 'FLW_123';
await initializePayment({ tx_ref: txRef, ... });
// Later...
await initializePayment({ tx_ref: txRef, ... });  // Error!

// CORRECT - unique reference per transaction
function generateTxRef(): string {
  const timestamp = Date.now().toString(36);
  const random = crypto.randomBytes(4).toString('hex');
  return `FLW_${timestamp}_${random}`.toUpperCase();
}
```

### Amount Conversion Errors

**Problem:** Payment amount is wrong (100x too small or too large)

```typescript
// WRONG - amount not in smallest unit for API
const payment = await initializePayment({
  amount: 5000,  // This might be interpreted differently
  ...
});

// CORRECT - be explicit about units
function toSmallestUnit(amount: number): number {
  return Math.round(amount * 100);
}

const payment = await initializePayment({
  amount: toSmallestUnit(5000),  // Clearly 5000 NGN = 500000 kobo
  ...
});
```

---

## API Issues

### Authentication Errors

**Problem:** `Invalid API key` or 401 Unauthorized

```typescript
// Check 1: Correct header format
const headers = {
  Authorization: `Bearer ${process.env.FLW_SECRET_KEY}`,  // Note: Bearer with space
  'Content-Type': 'application/json',
};

// Check 2: Key format (FLWSECK_TEST-xxx or FLWSECK-xxx)
console.log('Key prefix:', process.env.FLW_SECRET_KEY?.substring(0, 12));
// Test: FLWSECK_TEST
// Live: FLWSECK-

// Check 3: Environment variable is set
if (!process.env.FLW_SECRET_KEY) {
  throw new Error('FLW_SECRET_KEY not set');
}

// Check 4: No extra whitespace
const key = process.env.FLW_SECRET_KEY.trim();
```

### Test vs Live Mode

**Problem:** Transactions not appearing, or test cards rejected

```typescript
// Test mode keys
FLW_SECRET_KEY=FLWSECK_TEST-xxx    // Contains _TEST
FLW_PUBLIC_KEY=FLWPUBK_TEST-xxx    // Contains _TEST

// Live mode keys
FLW_SECRET_KEY=FLWSECK-xxx         // No _TEST
FLW_PUBLIC_KEY=FLWPUBK-xxx         // No _TEST

// Test cards only work with test keys
// Real cards only work with live keys
```

### Network Errors

**Problem:** `ECONNREFUSED`, `ETIMEDOUT`, or network errors

```typescript
// Add timeout and retry logic
async function flutterwaveRequest(
  url: string,
  options: RequestInit,
  retries = 3
) {
  const controller = new AbortController();
  const timeout = setTimeout(() => controller.abort(), 30000);

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
      return flutterwaveRequest(url, options, retries - 1);
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
   curl -X POST https://yourdomain.com/api/flutterwave/webhook \
     -H "Content-Type: application/json" \
     -d '{"test": true}'
   ```

2. **HTTPS is required** (for production)
   - Flutterwave only sends webhooks to HTTPS URLs
   - Use ngrok for local development

3. **Check Flutterwave Dashboard**
   - Go to Settings → Webhooks
   - View delivery logs
   - Check for failed deliveries

4. **Firewall/Security rules**
   - Ensure POST requests are allowed
   - Check WAF rules

### Signature Verification Failing

**Problem:** `Invalid signature` even with correct secret hash

```typescript
// COMMON MISTAKE 1: Wrong header name
// verif-hash (simple method) vs flutterwave-signature (HMAC method)

// Using verif-hash (simple)
const signature = req.headers['verif-hash'];
if (signature !== process.env.FLW_SECRET_HASH) {
  return res.status(401).json({ error: 'Invalid signature' });
}

// COMMON MISTAKE 2: Extra whitespace or case issues
const secretHash = process.env.FLW_SECRET_HASH?.trim();
if (signature?.trim() !== secretHash) { ... }

// COMMON MISTAKE 3: Using parsed JSON for HMAC
// WRONG
const isValid = verifyHMAC(JSON.stringify(req.body), signature);

// CORRECT - use raw body
const rawBody = await request.text();
const isValid = verifyHMAC(rawBody, signature);
```

### Webhook Timeout

**Problem:** Flutterwave marks webhook as failed

```typescript
// WRONG - Heavy processing before response
app.post('/webhook', async (req, res) => {
  await updateDatabase();
  await sendEmails();
  await processOrder();
  res.json({ ok: true });  // Too late!
});

// CORRECT - Respond immediately, process async
app.post('/webhook', async (req, res) => {
  res.json({ received: true });  // Respond immediately

  // Process in background
  processWebhookAsync(req.body).catch(console.error);
});
```

---

## Payment Issues

### Payment Not Redirecting

**Problem:** User sees error on payment page

1. **Check redirect_url** - Must be valid HTTPS URL in production
2. **Check amount** - Must be positive number in smallest unit
3. **Check currency** - Must be supported (NGN, GHS, KES, etc.)
4. **Check customer email** - Must be valid format

### Payment Successful But Not Verified

**Problem:** User paid but your system shows unpaid

```typescript
// 1. Don't trust callback parameters alone
// Callback URL can be manipulated

// 2. ALWAYS verify server-side
async function handleCallback(transactionId: number, txRef: string) {
  // Don't trust: status param in URL
  // Do verify: with Flutterwave API
  const transaction = await verifyTransaction(transactionId);

  if (transaction.status === 'successful') {
    // Now safe to fulfill
  }
}

// 3. Use webhooks as backup
// charge.completed webhook will arrive even if callback fails

// 4. Verify response thoroughly
if (transaction.status !== 'successful') {
  // Payment not complete - could be pending, failed, etc.
  return;
}

// 5. CRITICAL: Verify amount
if (transaction.amount !== expectedAmount) {
  // Amount mismatch - possible fraud!
  return;
}
```

### Transaction Status Confusion

**Problem:** `success-pending-validation` status

```typescript
// This status is common for bank transfers and some mobile money
if (transaction.status === 'success-pending-validation') {
  // DON'T fulfill yet!
  // Wait for webhook with final status

  // Option 1: Poll for status
  // Option 2: Wait for webhook (recommended)
}

// Status meanings:
// successful - Payment complete, safe to fulfill
// pending - Still processing, wait
// success-pending-validation - Needs validation, wait for webhook
// failed - Payment failed
```

---

## Testing

### Test Cards

| Card Number | CVV | Expiry | PIN | OTP | Result |
|-------------|-----|--------|-----|-----|--------|
| `5531886652142950` | 564 | 09/32 | 3310 | 12345 | Success |
| `5258585922666506` | 883 | 09/31 | 3310 | 12345 | Insufficient Funds |
| `5399838383838381` | 470 | 10/31 | 3310 | 12345 | Declined |

### Test Mobile Money

| Country | Phone Number | OTP |
|---------|--------------|-----|
| Ghana | 0551234987 | 123456 |
| Kenya | 254712345678 | Auto-approved |
| Uganda | 256772123456 | 123456 |

### Testing Webhooks Locally

```bash
# 1. Start your server
npm run dev

# 2. Install and run ngrok
ngrok http 3000

# 3. Set webhook URL in Flutterwave Dashboard
# Settings → Webhooks → URL: https://abc123.ngrok.io/api/flutterwave/webhook

# 4. Set your secret hash in Dashboard
# Copy it to your .env as FLW_SECRET_HASH

# 5. Make a test payment to trigger webhook

# 6. View ngrok logs
# Open http://localhost:4040
```

### Manual Webhook Testing

```bash
# Test with verif-hash method
SECRET_HASH="your_secret_hash"
PAYLOAD='{"event":"charge.completed","data":{"id":123,"tx_ref":"TEST_123","amount":500000,"status":"successful","customer":{"email":"test@example.com"}}}'

curl -X POST http://localhost:3000/api/flutterwave/webhook \
  -H "Content-Type: application/json" \
  -H "verif-hash: $SECRET_HASH" \
  -d "$PAYLOAD"
```

### Testing Checklist

- [ ] Initialize payment with valid data
- [ ] Initialize payment with missing email (expect error)
- [ ] Initialize payment with invalid amount (expect error)
- [ ] Complete payment with test card
- [ ] Verify successful payment
- [ ] Verify failed payment status
- [ ] Receive and process charge.completed webhook
- [ ] Verify webhook signature validation
- [ ] Reject webhook with invalid signature
- [ ] Test mobile money flow
- [ ] Test bank transfer flow

---

## Debug Checklist

### API Calls Failing?

1. **Check Authorization header**
   ```typescript
   Authorization: `Bearer ${FLW_SECRET_KEY}`
   // NOT: Bearer${key} (missing space)
   // NOT: Basic ${key}
   ```

2. **Verify base URL**
   ```typescript
   const BASE_URL = 'https://api.flutterwave.com/v3';
   // NOT: https://api.flutterwave.com (missing version)
   ```

3. **Check key mode**
   - Test keys: `FLWSECK_TEST-xxx`, `FLWPUBK_TEST-xxx`
   - Live keys: `FLWSECK-xxx`, `FLWPUBK-xxx`

4. **Enable request logging**
   ```typescript
   async function debugFetch(url: string, options: RequestInit) {
     console.log('Request:', { url, method: options.method });
     const response = await fetch(url, options);
     const data = await response.json();
     console.log('Response:', { status: response.status, data });
     return data;
   }
   ```

### Webhook Not Working?

1. **Check URL is accessible**
   ```bash
   curl -X POST https://yourdomain.com/webhook -d '{}' -v
   ```

2. **Check signature configuration**
   - Dashboard secret hash matches env var
   - Using correct header (verif-hash or flutterwave-signature)

3. **Check response time**
   - Return 200 within 30 seconds
   - Process heavy work asynchronously

4. **Check body parsing**
   - For HMAC: Use raw body before JSON parsing
   - For verif-hash: Can parse JSON normally

### Payment Issues?

1. **Verify amount calculation**
   ```typescript
   // NGN 5000 = 500000 kobo
   const amountInKobo = amount * 100;
   ```

2. **Check currency**
   - NGN, GHS, KES, ZAR, USD, XOF, XAF

3. **Verify callback URL**
   - Must be HTTPS in production
   - Must be publicly accessible

4. **Check status handling**
   ```typescript
   // Handle all statuses
   switch (transaction.status) {
     case 'successful': // Fulfill
     case 'pending': // Wait
     case 'success-pending-validation': // Wait for webhook
     case 'failed': // Show error
   }
   ```

---

## Debugging Tools

### Enable Verbose Logging

```typescript
// Log all Flutterwave requests
async function flutterwaveFetch(url: string, options: RequestInit) {
  console.log('Flutterwave Request:', {
    url,
    method: options.method,
    body: options.body ? JSON.parse(options.body as string) : undefined,
  });

  const response = await fetch(url, options);
  const data = await response.json();

  console.log('Flutterwave Response:', {
    status: response.status,
    data,
  });

  return data;
}
```

### Check Flutterwave Status

If experiencing widespread issues:
- [Flutterwave Status](https://status.flutterwave.com/)
- [Flutterwave Twitter](https://twitter.com/theaborode)

### Contact Support

For unresolved issues:
- Email: developers@flutterwave.com
- Include: Transaction reference, timestamp, error message, request/response logs
