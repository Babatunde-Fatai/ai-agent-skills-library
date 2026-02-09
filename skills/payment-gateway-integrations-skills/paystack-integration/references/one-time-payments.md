> **When to read:** When implementing frontend popup/redirect or initialize/verify one-time payments.
> **What problem it solves:** Shows minimal flow for initializing, collecting, and verifying single payments.
> **When to skip:** If you only work on backend webhooks or subscriptions.
> **Prerequisites:** Read AGENT_EXECUTION_SPEC.md first.

# One-Time Payments - Complete Guide

This guide covers the complete one-time payment flow: initialization, payment collection (redirect or popup), and verification.

## Table of Contents

1. [Payment Flow Overview](#payment-flow-overview)
2. [Initialize Transaction](#initialize-transaction)
3. [Collect Payment](#collect-payment)
4. [Verify Transaction](#verify-transaction)
5. [Charge Authorization](#charge-authorization)
6. [Split Payments](#split-payments)

---


## Payment Flow Overview

```
┌──────────────────────────────────────────────────────────────────────────┐
│                         ONE-TIME PAYMENT FLOW                            │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. INITIALIZE                                                           │
│     POST /transaction/initialize                                         │
│     → Returns: authorization_url, access_code, reference                 │
│                                                                          │
│  2. COLLECT PAYMENT (choose one)                                         │
│     ┌─────────────────────┐    ┌─────────────────────┐                  │
│     │  REDIRECT FLOW      │    │  POPUP FLOW         │                  │
│     │  Redirect user to   │    │  Use Inline.js with │                  │
│     │  authorization_url  │    │  access_code        │                  │
│     │  User pays on       │    │  User pays in popup │                  │
│     │  Paystack page      │    │  on your site       │                  │
│     └─────────────────────┘    └─────────────────────┘                  │
│                                                                          │
│  3. VERIFY                                                               │
│     GET /transaction/verify/:reference                                   │
│     → Confirm status === 'success' AND amount matches                    │
│                                                                          │
│  4. FULFILL ORDER                                                        │
│     Update database, send confirmation, deliver product                  │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## Initialize Transaction

### API Endpoint

```
POST https://api.paystack.co/transaction/initialize
```

### Request Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `email` | string | Yes | Customer email |
| `amount` | integer | Yes | Amount in kobo (NGN) or pesewas (GHS) |
| `reference` | string | No | Unique transaction reference (auto-generated if omitted) |
| `callback_url` | string | No | URL to redirect after payment |
| `metadata` | object | No | Custom data (order ID, user ID, etc.) |
| `channels` | array | No | Payment channels: `['card', 'bank', 'ussd', 'qr', 'mobile_money', 'bank_transfer']` |
| `currency` | string | No | Currency code (NGN, GHS, ZAR, KES). Defaults to integration currency |
| `plan` | string | No | Plan code for subscriptions |
| `subaccount` | string | No | Subaccount code for split payments |
| `split_code` | string | No | Multi-split configuration code |

### TypeScript Types

```typescript
interface InitializeTransactionRequest {
  email: string;
  amount: number;
  reference?: string;
  callback_url?: string;
  metadata?: {
    custom_fields?: Array<{
      display_name: string;
      variable_name: string;
      value: string | number;
    }>;
    [key: string]: unknown;
  };
  channels?: Array<'card' | 'bank' | 'ussd' | 'qr' | 'mobile_money' | 'bank_transfer' | 'eft'>;
  currency?: 'NGN' | 'GHS' | 'ZAR' | 'KES' | 'USD';
  plan?: string;
  subaccount?: string;
  split_code?: string;
  transaction_charge?: number;
  bearer?: 'account' | 'subaccount';
}

interface InitializeTransactionResponse {
  status: boolean;
  message: string;
  data: {
    authorization_url: string;
    access_code: string;
    reference: string;
  };
}
```

### Implementation

```typescript
const PAYSTACK_BASE_URL = 'https://api.paystack.co';

async function initializeTransaction(
  params: InitializeTransactionRequest
): Promise<InitializeTransactionResponse['data']> {
  // Validate amount
  if (params.amount < 100) {
    throw new Error('Minimum amount is 100 kobo (1 NGN)');
  }

  const response = await fetch(`${PAYSTACK_BASE_URL}/transaction/initialize`, {
    method: 'POST',
    headers: {
      Authorization: `Bearer ${process.env.PAYSTACK_SECRET_KEY}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify(params),
  });

  const result: InitializeTransactionResponse = await response.json();

  if (!result.status) {
    throw new Error(`Paystack error: ${result.message}`);
  }

  return result.data;
}

// Usage example
const transaction = await initializeTransaction({
  email: 'customer@example.com',
  amount: 500000, // 5,000 NGN in kobo
  callback_url: 'https://yoursite.com/payment/callback',
  metadata: {
    order_id: 'ORD-123',
    custom_fields: [
      {
        display_name: 'Order ID',
        variable_name: 'order_id',
        value: 'ORD-123',
      },
    ],
  },
  channels: ['card', 'bank_transfer'],
});

console.log(transaction.authorization_url); // Redirect user here
console.log(transaction.access_code);       // Or use with Inline.js
console.log(transaction.reference);         // Store for verification
```

### Generating Unique References

```typescript
import crypto from 'crypto';

function generateReference(prefix = 'TXN'): string {
  const timestamp = Date.now().toString(36);
  const randomPart = crypto.randomBytes(4).toString('hex');
  return `${prefix}_${timestamp}_${randomPart}`.toUpperCase();
}

// Examples: TXN_LK5J2M8_A1B2C3D4, TXN_LK5J2M9_E5F6G7H8
```

---

## Collect Payment

### Option A: Redirect Flow

Simple approach - redirect user to Paystack's hosted payment page.

```typescript
// After initializing transaction
const { authorization_url, reference } = await initializeTransaction({
  email: 'customer@example.com',
  amount: 500000,
  callback_url: 'https://yoursite.com/payment/verify',
});

// Store reference in session/database for verification
await saveTransactionReference(reference, orderId);

// Redirect user (in API response)
return Response.redirect(authorization_url);

// Or return URL to frontend
return Response.json({ redirectUrl: authorization_url });
```

**Frontend redirect:**
```typescript
// React/Next.js
window.location.href = authorization_url;

// Or with router
router.push(authorization_url);
```

### Option B: Popup Flow (Inline.js)

Better UX - payment happens in a popup on your site.

**Install SDK:**
```bash
npm install @paystack/inline-js
```

**Frontend Component (React/Next.js):**
```typescript
'use client';

import { useCallback } from 'react';
import PaystackPop from '@paystack/inline-js';

interface PaystackButtonProps {
  email: string;
  amount: number; // in Naira (will be converted to kobo)
  onSuccess: (reference: string) => void;
  onCancel: () => void;
  metadata?: Record<string, unknown>;
}

export function PaystackButton({
  email,
  amount,
  onSuccess,
  onCancel,
  metadata,
}: PaystackButtonProps) {
  const handlePayment = useCallback(async () => {
    // 1. Initialize transaction on backend
    const response = await fetch('/api/paystack/initialize', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email, amount: amount * 100, metadata }),
    });

    const { access_code, reference } = await response.json();

    // 2. Open popup with access_code
    const popup = new PaystackPop();
    popup.newTransaction({
      key: process.env.NEXT_PUBLIC_PAYSTACK_PUBLIC_KEY!,
      accessCode: access_code,
      onSuccess: (transaction) => {
        console.log('Payment successful:', transaction);
        onSuccess(reference);
      },
      onCancel: () => {
        console.log('Payment cancelled');
        onCancel();
      },
    });
  }, [email, amount, metadata, onSuccess, onCancel]);

  return (
    <button
      onClick={handlePayment}
      className="bg-green-600 text-white px-6 py-3 rounded-lg"
    >
      Pay ₦{amount.toLocaleString()}
    </button>
  );
}
```

**Alternative: Direct with public key (simpler but less secure):**
```typescript
// Only use for simple cases - initializing on backend is more secure
const popup = new PaystackPop();
popup.newTransaction({
  key: process.env.NEXT_PUBLIC_PAYSTACK_PUBLIC_KEY!,
  email: 'customer@example.com',
  amount: 500000, // kobo
  onSuccess: (transaction) => {
    // Verify on backend
    verifyPayment(transaction.reference);
  },
  onCancel: () => {
    console.log('Cancelled');
  },
});
```

---

## Verify Transaction

**CRITICAL: Always verify transactions server-side. Never trust client-side callbacks alone.**

### API Endpoint

```
GET https://api.paystack.co/transaction/verify/:reference
```

### TypeScript Types

```typescript
interface VerifyTransactionResponse {
  status: boolean;
  message: string;
  data: {
    id: number;
    domain: 'test' | 'live';
    status: 'success' | 'failed' | 'abandoned' | 'pending';
    reference: string;
    amount: number;
    currency: string;
    channel: string;
    gateway_response: string;
    ip_address: string;
    fees: number | null;
    authorization: {
      authorization_code: string;
      bin: string;
      last4: string;
      exp_month: string;
      exp_year: string;
      channel: string;
      card_type: string;
      bank: string;
      country_code: string;
      brand: string;
      reusable: boolean;
      signature: string;
      account_name: string | null;
    } | null;
    customer: {
      id: number;
      first_name: string | null;
      last_name: string | null;
      email: string;
      customer_code: string;
      phone: string | null;
      metadata: Record<string, unknown> | null;
    };
    plan: string | null;
    plan_object: Record<string, unknown> | null;
    metadata: Record<string, unknown> | null;
    paid_at: string | null;
    created_at: string;
    requested_amount: number;
    transaction_date: string;
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
  reference: string,
  expectedAmount: number
): Promise<VerificationResult> {
  try {
    const response = await fetch(
      `${PAYSTACK_BASE_URL}/transaction/verify/${encodeURIComponent(reference)}`,
      {
        headers: {
          Authorization: `Bearer ${process.env.PAYSTACK_SECRET_KEY}`,
        },
      }
    );

    const result: VerifyTransactionResponse = await response.json();

    if (!result.status) {
      return {
        success: false,
        transaction: null,
        error: result.message,
      };
    }

    const { data } = result;

    // CHECK 1: Verify payment status
    if (data.status !== 'success') {
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

// Usage
const result = await verifyTransaction('TXN_ABC123', 500000);

if (result.success) {
  // Fulfill order
  await fulfillOrder(orderId, result.transaction!);
} else {
  // Handle failure
  console.error('Payment verification failed:', result.error);
}
```

### Callback URL Handler

```typescript
// /api/payment/callback or /payment/verify page

export async function GET(request: Request) {
  const url = new URL(request.url);
  const reference = url.searchParams.get('reference');
  const trxref = url.searchParams.get('trxref'); // Alternative param

  const ref = reference || trxref;

  if (!ref) {
    return Response.redirect('/payment/error?reason=missing_reference');
  }

  // Get expected amount from your database
  const order = await getOrderByReference(ref);

  if (!order) {
    return Response.redirect('/payment/error?reason=order_not_found');
  }

  const result = await verifyTransaction(ref, order.amountInKobo);

  if (result.success) {
    await markOrderAsPaid(order.id);
    return Response.redirect(`/payment/success?order=${order.id}`);
  } else {
    return Response.redirect(`/payment/failed?reason=${encodeURIComponent(result.error || 'unknown')}`);
  }
}
```

---

## Charge Authorization

Charge a previously authorized card without customer intervention. Useful for:
- Recurring charges outside of subscriptions
- Saving card for future purchases
- One-click checkout

### Save Authorization

After successful payment, save the authorization code:

```typescript
// From verification response
const { authorization } = verificationResult.transaction!;

if (authorization?.reusable) {
  await saveCustomerCard({
    customerId: customer.id,
    authorizationCode: authorization.authorization_code,
    last4: authorization.last4,
    cardType: authorization.card_type,
    bank: authorization.bank,
    expMonth: authorization.exp_month,
    expYear: authorization.exp_year,
  });
}
```

### Charge Saved Card

```
POST https://api.paystack.co/transaction/charge_authorization
```

```typescript
interface ChargeAuthorizationRequest {
  authorization_code: string;
  email: string;
  amount: number;
  reference?: string;
  currency?: string;
  metadata?: Record<string, unknown>;
}

async function chargeAuthorization(
  params: ChargeAuthorizationRequest
): Promise<VerifyTransactionResponse['data']> {
  const response = await fetch(
    `${PAYSTACK_BASE_URL}/transaction/charge_authorization`,
    {
      method: 'POST',
      headers: {
        Authorization: `Bearer ${process.env.PAYSTACK_SECRET_KEY}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify(params),
    }
  );

  const result = await response.json();

  if (!result.status) {
    throw new Error(result.message);
  }

  return result.data;
}

// Usage
const charge = await chargeAuthorization({
  authorization_code: 'AUTH_xxx',
  email: 'customer@example.com',
  amount: 250000, // 2,500 NGN
  reference: generateReference('CHG'),
});

if (charge.status === 'success') {
  console.log('Charge successful');
}
```

---

## Split Payments

Split payments between your main account and subaccounts (for marketplaces, affiliates, etc.).

### Create Subaccount

```
POST https://api.paystack.co/subaccount
```

```typescript
interface CreateSubaccountRequest {
  business_name: string;
  settlement_bank: string; // Bank code
  account_number: string;
  percentage_charge: number; // Subaccount's share (0-100)
  description?: string;
  primary_contact_email?: string;
  primary_contact_name?: string;
  primary_contact_phone?: string;
  metadata?: Record<string, unknown>;
}

async function createSubaccount(
  params: CreateSubaccountRequest
): Promise<{ subaccount_code: string }> {
  const response = await fetch(`${PAYSTACK_BASE_URL}/subaccount`, {
    method: 'POST',
    headers: {
      Authorization: `Bearer ${process.env.PAYSTACK_SECRET_KEY}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify(params),
  });

  const result = await response.json();

  if (!result.status) {
    throw new Error(result.message);
  }

  return result.data;
}

// Usage
const subaccount = await createSubaccount({
  business_name: 'Vendor Store',
  settlement_bank: '058', // GTBank code
  account_number: '0123456789',
  percentage_charge: 80, // Vendor gets 80%
});
```

### Split Payment on Transaction

```typescript
// Single split (one subaccount)
const transaction = await initializeTransaction({
  email: 'customer@example.com',
  amount: 100000, // 1,000 NGN
  subaccount: 'ACCT_xxx', // Subaccount code
  bearer: 'subaccount', // Who bears Paystack fees
  // transaction_charge: 10000, // Optional flat fee to main account
});

// Multi-split (multiple subaccounts)
// First create a split configuration in dashboard or via API
const transaction = await initializeTransaction({
  email: 'customer@example.com',
  amount: 100000,
  split_code: 'SPL_xxx', // Multi-split configuration
});
```

### Bearer Options

| Bearer | Description |
|--------|-------------|
| `account` | Main account bears fees |
| `subaccount` | Subaccount bears fees |

---

## Complete Flow Example

```typescript
// STEP 1: Call Paystack FIRST — let Paystack generate the reference
// Do NOT provide a reference parameter
const init = await initializeTransaction({
  email: 'buyer@example.com',
  amount: 750000, // 7,500 NGN
  callback_url: 'https://shop.com/verify',
  metadata: {
    order_id: 'ORD-456',
    items: ['Product A', 'Product B'],
  },
  // Note: reference is intentionally omitted — Paystack generates it
});

// STEP 2: Store order with Paystack's reference AFTER successful initialization
// CRITICAL: Use init.reference (from Paystack), not a locally-generated one
await db.orders.update({
  where: { id: orderId },
  data: {
    paymentReference: init.reference,  // ← Paystack's reference
    paymentStatus: 'pending',
  },
});

// 3. Redirect or popup (see Collect Payment section)

// 4. On callback/webhook, verify
const order = await db.orders.findUnique({
  where: { paymentReference: reference },
});

const result = await verifyTransaction(reference, order.totalInKobo);

if (result.success) {
  // 5. Fulfill
  await db.orders.update({
    where: { id: order.id },
    data: {
      paymentStatus: 'paid',
      paidAt: new Date(),
      authorizationCode: result.transaction?.authorization?.authorization_code,
    },
  });

  await sendOrderConfirmationEmail(order);
  await notifyFulfillmentTeam(order);
}
```
