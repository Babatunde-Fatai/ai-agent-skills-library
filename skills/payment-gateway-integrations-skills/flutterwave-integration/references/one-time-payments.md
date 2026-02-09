> **When to read:** When implementing frontend popup/redirect or initialize/verify one-time payments.
> **What problem it solves:** Shows minimal flow for initializing, collecting, and verifying single payments.
> **When to skip:** If you only work on backend webhooks or subscriptions.
> **Prerequisites:** Read AGENT_EXECUTION_SPEC.md first.

# One-Time Payments - Complete Guide

This guide covers the complete one-time payment flow: initialization, payment collection (redirect or inline), and verification.

## Table of Contents

1. [Payment Flow Overview](#payment-flow-overview)
2. [Initialize Transaction](#initialize-transaction)
3. [Collect Payment](#collect-payment)
4. [Verify Transaction](#verify-transaction)
5. [Charge Authorization (Tokenization)](#charge-authorization)
6. [Complete Flow Example](#complete-flow-example)

---

## Payment Flow Overview

```
┌──────────────────────────────────────────────────────────────────────────┐
│                         ONE-TIME PAYMENT FLOW                            │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. GENERATE & STORE                                                     │
│     Generate unique tx_ref                                               │
│     Store order in DB with status=pending                                │
│                                                                          │
│  2. INITIALIZE                                                           │
│     POST /v3/payments                                                    │
│     → Returns: link (hosted payment URL)                                 │
│                                                                          │
│  3. COLLECT PAYMENT (choose one)                                         │
│     ┌─────────────────────┐    ┌─────────────────────┐                  │
│     │  REDIRECT FLOW      │    │  INLINE FLOW        │                  │
│     │  Redirect user to   │    │  Use Flutterwave    │                  │
│     │  hosted link        │    │  Inline.js popup    │                  │
│     └─────────────────────┘    └─────────────────────┘                  │
│                                                                          │
│  4. VERIFY                                                               │
│     GET /v3/transactions/:id/verify                                      │
│     → Confirm status === 'successful' AND amount matches                 │
│                                                                          │
│  5. FULFILL ORDER                                                        │
│     Update database, send confirmation, deliver product                  │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## Initialize Transaction

### API Endpoint

```
POST https://api.flutterwave.com/v3/payments
```

### Request Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `tx_ref` | string | Yes | YOUR unique transaction reference |
| `amount` | number | Yes | Amount in smallest unit (kobo, pesewas, cents) |
| `currency` | string | Yes | Currency code (NGN, GHS, KES, ZAR, USD, etc.) |
| `redirect_url` | string | Yes | URL to redirect after payment |
| `customer.email` | string | Yes | Customer email |
| `customer.name` | string | No | Customer full name |
| `customer.phonenumber` | string | No | Customer phone number |
| `payment_options` | string | No | Comma-separated: "card,banktransfer,ussd,mobilemoney" |
| `customizations.title` | string | No | Payment page title |
| `customizations.description` | string | No | Payment description |
| `customizations.logo` | string | No | Logo URL |
| `meta` | object | No | Custom metadata |

### TypeScript Types

```typescript
interface InitializePaymentRequest {
  tx_ref: string;
  amount: number;
  currency: string;
  redirect_url: string;
  customer: {
    email: string;
    name?: string;
    phonenumber?: string;
  };
  payment_options?: string;
  customizations?: {
    title?: string;
    description?: string;
    logo?: string;
  };
  meta?: Record<string, unknown>;
}

interface InitializePaymentResponse {
  status: 'success' | 'error';
  message: string;
  data: {
    link: string;  // Hosted payment page URL
  };
}
```

### Implementation

```typescript
import crypto from 'crypto';

const FLW_BASE_URL = 'https://api.flutterwave.com/v3';

function getHeaders() {
  return {
    Authorization: `Bearer ${process.env.FLW_SECRET_KEY}`,
    'Content-Type': 'application/json',
  };
}

// Generate unique tx_ref
function generateTxRef(prefix = 'FLW'): string {
  const timestamp = Date.now().toString(36);
  const random = crypto.randomBytes(4).toString('hex');
  return `${prefix}_${timestamp}_${random}`.toUpperCase();
}

// Convert to smallest unit
function toSmallestUnit(amount: number): number {
  return Math.round(amount * 100);
}

// Initialize payment
async function initializePayment(
  params: InitializePaymentRequest
): Promise<string> {
  const response = await fetch(`${FLW_BASE_URL}/payments`, {
    method: 'POST',
    headers: getHeaders(),
    body: JSON.stringify(params),
  });

  const result: InitializePaymentResponse = await response.json();

  if (result.status !== 'success') {
    throw new Error(`Flutterwave error: ${result.message}`);
  }

  return result.data.link;
}

// Usage example
const txRef = generateTxRef('ORD');

// STEP 1: Store in database FIRST
// await db.orders.create({
//   data: {
//     txRef,
//     amount: toSmallestUnit(5000),  // 5000 NGN
//     currency: 'NGN',
//     status: 'pending',
//     customerEmail: 'customer@example.com',
//   },
// });

// STEP 2: Then call Flutterwave
const paymentLink = await initializePayment({
  tx_ref: txRef,
  amount: toSmallestUnit(5000),  // 500000 kobo
  currency: 'NGN',
  redirect_url: 'https://yoursite.com/payment/callback',
  customer: {
    email: 'customer@example.com',
    name: 'John Doe',
  },
  customizations: {
    title: 'My Store',
    description: 'Payment for Order #123',
  },
  meta: {
    order_id: 'ORD-123',
  },
});

console.log(paymentLink);  // Redirect user to this URL
```

---

## Collect Payment

### Option A: Redirect Flow (Recommended)

Simple approach - redirect user to Flutterwave's hosted payment page.

```typescript
// After initializing transaction
const { txRef, paymentLink } = await initializePayment({...});

// Redirect user (in API response)
return Response.redirect(paymentLink);

// Or return URL to frontend
return Response.json({ redirectUrl: paymentLink, txRef });
```

**Frontend redirect:**
```typescript
// React/Next.js
window.location.href = paymentLink;

// Or with router
router.push(paymentLink);
```

### Option B: Inline Flow (Popup)

Better UX - payment happens in a popup on your site.

**Install SDK:**
```bash
npm install flutterwave-react-v3
# Or include script in HTML
```

**HTML Script Include:**
```html
<script src="https://checkout.flutterwave.com/v3.js"></script>
```

**Frontend Component (React):**
```typescript
'use client';

import { useFlutterwave, closePaymentModal } from 'flutterwave-react-v3';

interface FlutterwaveButtonProps {
  txRef: string;
  email: string;
  amount: number;  // In naira/cedi/etc (not kobo)
  currency: string;
  onSuccess: (response: FlutterwaveResponse) => void;
  onClose: () => void;
}

interface FlutterwaveResponse {
  status: string;
  transaction_id: number;
  tx_ref: string;
  flw_ref: string;
}

export function FlutterwaveButton({
  txRef,
  email,
  amount,
  currency,
  onSuccess,
  onClose,
}: FlutterwaveButtonProps) {
  const config = {
    public_key: process.env.NEXT_PUBLIC_FLW_PUBLIC_KEY!,
    tx_ref: txRef,
    amount: amount,  // Inline JS expects display amount, not kobo
    currency: currency,
    payment_options: 'card,mobilemoney,ussd,banktransfer',
    customer: {
      email: email,
    },
    customizations: {
      title: 'My Store',
      description: 'Payment for your order',
      logo: 'https://yoursite.com/logo.png',
    },
  };

  const handleFlutterPayment = useFlutterwave(config);

  return (
    <button
      onClick={() => {
        handleFlutterPayment({
          callback: (response) => {
            console.log('Payment response:', response);
            closePaymentModal();

            if (response.status === 'successful') {
              onSuccess(response);
            }
          },
          onClose: () => {
            console.log('Payment modal closed');
            onClose();
          },
        });
      }}
      className="bg-orange-500 text-white px-6 py-3 rounded-lg"
    >
      Pay {currency} {amount.toLocaleString()}
    </button>
  );
}
```

**Vanilla JavaScript:**
```javascript
function makePayment(txRef, email, amount, currency) {
  FlutterwaveCheckout({
    public_key: 'FLWPUBK_TEST-xxx',
    tx_ref: txRef,
    amount: amount,
    currency: currency,
    payment_options: 'card,mobilemoney,ussd,banktransfer',
    customer: {
      email: email,
    },
    callback: function(response) {
      console.log('Payment response:', response);
      // Verify on backend
      verifyPayment(response.transaction_id);
    },
    onclose: function() {
      console.log('Payment modal closed');
    },
  });
}
```

---

## Verify Transaction

**CRITICAL: Always verify transactions server-side. Never trust client-side callbacks alone.**

### Verification Endpoints

```
# By Transaction ID (recommended)
GET https://api.flutterwave.com/v3/transactions/{id}/verify

# By tx_ref
GET https://api.flutterwave.com/v3/transactions/verify_by_reference?tx_ref={tx_ref}
```

### TypeScript Types

```typescript
interface VerifyTransactionResponse {
  status: 'success' | 'error';
  message: string;
  data: {
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
    status: 'successful' | 'pending' | 'failed' | 'success-pending-validation';
    payment_type: string;
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
  };
}
```

### Implementation

```typescript
interface VerificationResult {
  success: boolean;
  transaction: VerifyTransactionResponse['data'] | null;
  error?: string;
}

async function verifyTransaction(
  transactionId: number,
  expectedAmount: number,
  expectedCurrency: string
): Promise<VerificationResult> {
  try {
    const response = await fetch(
      `${FLW_BASE_URL}/transactions/${transactionId}/verify`,
      { headers: getHeaders() }
    );

    const result: VerifyTransactionResponse = await response.json();

    if (result.status !== 'success') {
      return {
        success: false,
        transaction: null,
        error: result.message,
      };
    }

    const { data } = result;

    // CHECK 1: Verify payment status
    if (data.status === 'pending' || data.status === 'success-pending-validation') {
      return {
        success: false,
        transaction: data,
        error: `Payment pending: ${data.status}. Wait for webhook.`,
      };
    }

    if (data.status !== 'successful') {
      return {
        success: false,
        transaction: data,
        error: `Payment not successful. Status: ${data.status}`,
      };
    }

    // CHECK 2: Verify amount (CRITICAL - prevents underpayment fraud)
    if (data.amount !== expectedAmount) {
      return {
        success: false,
        transaction: data,
        error: `Amount mismatch. Expected: ${expectedAmount}, Got: ${data.amount}`,
      };
    }

    // CHECK 3: Verify currency
    if (data.currency !== expectedCurrency) {
      return {
        success: false,
        transaction: data,
        error: `Currency mismatch. Expected: ${expectedCurrency}, Got: ${data.currency}`,
      };
    }

    return {
      success: true,
      transaction: data,
    };
  } catch (error) {
    return {
      success: false,
      transaction: null,
      error: error instanceof Error ? error.message : 'Verification failed',
    };
  }
}

// Verify by tx_ref (alternative)
async function verifyByReference(
  txRef: string,
  expectedAmount: number,
  expectedCurrency: string
): Promise<VerificationResult> {
  const response = await fetch(
    `${FLW_BASE_URL}/transactions/verify_by_reference?tx_ref=${encodeURIComponent(txRef)}`,
    { headers: getHeaders() }
  );

  const result: VerifyTransactionResponse = await response.json();
  // ... same verification logic as above
}
```

### Callback URL Handler

```typescript
// /api/payment/callback or /payment/verify page

export async function GET(request: Request) {
  const url = new URL(request.url);
  const status = url.searchParams.get('status');
  const txRef = url.searchParams.get('tx_ref');
  const transactionId = url.searchParams.get('transaction_id');

  if (!txRef || !transactionId) {
    return Response.redirect('/payment/error?reason=missing_params');
  }

  // Get expected amount from your database
  const order = await getOrderByTxRef(txRef);

  if (!order) {
    return Response.redirect('/payment/error?reason=order_not_found');
  }

  // CRITICAL: Don't trust status param, verify with Flutterwave
  const result = await verifyTransaction(
    parseInt(transactionId),
    order.amountInSmallestUnit,
    order.currency
  );

  if (result.success) {
    await markOrderAsPaid(order.id, result.transaction!);
    return Response.redirect(`/payment/success?order=${order.id}`);
  } else {
    return Response.redirect(`/payment/failed?reason=${encodeURIComponent(result.error || 'unknown')}`);
  }
}
```

---

## Charge Authorization (Tokenization)

Save card for future purchases without collecting card details again.

### Save Card Token After Successful Payment

```typescript
// From verification response after successful payment
const { data } = verificationResult;

if (data.card) {
  // Flutterwave returns a token in subsequent transactions
  // You can charge the card again using the token
  await saveCustomerCard({
    customerId: customer.id,
    cardToken: data.card.token,  // Available in some responses
    last4: data.card.last_4digits,
    cardType: data.card.type,
    issuer: data.card.issuer,
    expiry: data.card.expiry,
  });
}
```

### Charge Saved Token

```
POST https://api.flutterwave.com/v3/tokenized-charges
```

```typescript
interface TokenizedChargeRequest {
  token: string;
  currency: string;
  country: string;
  amount: number;
  email: string;
  tx_ref: string;
  narration?: string;
}

async function chargeToken(
  params: TokenizedChargeRequest
): Promise<VerifyTransactionResponse['data']> {
  const response = await fetch(`${FLW_BASE_URL}/tokenized-charges`, {
    method: 'POST',
    headers: getHeaders(),
    body: JSON.stringify(params),
  });

  const result = await response.json();

  if (result.status !== 'success') {
    throw new Error(result.message);
  }

  return result.data;
}

// Usage
const charge = await chargeToken({
  token: 'flw-t1nf-xxx',
  currency: 'NGN',
  country: 'NG',
  amount: toSmallestUnit(2500),  // 2,500 NGN
  email: 'customer@example.com',
  tx_ref: generateTxRef('TOK'),
});
```

---

## Complete Flow Example

```typescript
// STEP 1: Generate unique tx_ref
const txRef = generateTxRef('ORD');
const amountInKobo = toSmallestUnit(7500);  // 7,500 NGN

// STEP 2: Store order in DB FIRST (before calling Flutterwave)
await db.orders.create({
  data: {
    txRef,
    amount: amountInKobo,
    currency: 'NGN',
    status: 'pending',
    customerEmail: 'buyer@example.com',
    metadata: {
      items: ['Product A', 'Product B'],
    },
  },
});

// STEP 3: Initialize payment with Flutterwave
const paymentLink = await initializePayment({
  tx_ref: txRef,
  amount: amountInKobo,
  currency: 'NGN',
  redirect_url: 'https://shop.com/payment/callback',
  customer: {
    email: 'buyer@example.com',
    name: 'John Doe',
  },
  meta: {
    order_id: 'ORD-456',
  },
});

// STEP 4: Return link to frontend for redirect
return Response.json({
  success: true,
  paymentLink,
  txRef,
});

// STEP 5: On callback, verify the payment
// (In callback handler)
const order = await db.orders.findUnique({
  where: { txRef },
});

const result = await verifyTransaction(
  transactionId,
  order.amount,
  order.currency
);

if (result.success) {
  // STEP 6: Fulfill order
  await db.orders.update({
    where: { id: order.id },
    data: {
      status: 'paid',
      paidAt: new Date(),
      transactionId: result.transaction!.id,
      flwRef: result.transaction!.flw_ref,
    },
  });

  await sendOrderConfirmationEmail(order);
  await notifyFulfillmentTeam(order);
}
```

---

## Transaction Status Reference

| Status | Meaning | Action |
|--------|---------|--------|
| `successful` | Payment completed | Fulfill order |
| `pending` | Awaiting completion | Poll or wait for webhook |
| `success-pending-validation` | Bank transfer/ACH pending | Wait for webhook |
| `failed` | Payment failed | Show error, allow retry |

**Important:** Always handle `success-pending-validation` - this status is common for bank transfers and ACH payments. Wait for the `charge.completed` webhook before fulfilling.
