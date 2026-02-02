> **When to read:** When implementing Paystack routes with Next.js App Router.
> **What problem it solves:** Shows file placement and raw-body webhook handling patterns for Next.js.
> **When to skip:** Not needed for non-Next.js projects.
> **Prerequisites:** Read AGENT_EXECUTION_SPEC.md first.

# Next.js Implementation - Complete Guide

Complete Paystack integration for Next.js 13+ App Router with TypeScript.

## Table of Contents

1. [Project Structure](#project-structure)
2. [Environment Setup](#environment-setup)
3. [Utility Functions](#utility-functions)
4. [API Routes](#api-routes)
5. [Client Components](#client-components)
6. [Webhook Handler](#webhook-handler)
7. [Complete Example](#complete-example)

---

## Project Structure

```
your-nextjs-app/
├── app/
│   ├── api/
│   │   └── paystack/
│   │       ├── initialize/
│   │       │   └── route.ts       # POST - Initialize payment
│   │       ├── verify/
│   │       │   └── route.ts       # GET - Verify payment
│   │       ├── webhook/
│   │       │   └── route.ts       # POST - Handle webhooks
│   │       └── subscription/
│   │           └── route.ts       # POST - Create subscription
│   ├── payment/
│   │   ├── page.tsx               # Payment page
│   │   ├── success/
│   │   │   └── page.tsx           # Success page
│   │   └── callback/
│   │       └── page.tsx           # Callback handler
│   └── layout.tsx
├── components/
│   └── PaystackButton.tsx         # Payment button component
├── lib/
│   └── paystack.ts                # Utility functions
├── types/
│   └── paystack.ts                # TypeScript types
└── .env.local
```

---

## Environment Setup

### .env.local

```bash
# Paystack Keys
PAYSTACK_SECRET_KEY=sk_test_xxxxxxxxxxxxxxxxxxxxxxxxxxxx
NEXT_PUBLIC_PAYSTACK_PUBLIC_KEY=pk_test_xxxxxxxxxxxxxxxxxxxxxxxxxxxx

# App URL (for callbacks)
NEXT_PUBLIC_APP_URL=http://localhost:3000
```

### .env.production

```bash
PAYSTACK_SECRET_KEY=sk_live_xxxxxxxxxxxxxxxxxxxxxxxxxxxx
NEXT_PUBLIC_PAYSTACK_PUBLIC_KEY=pk_live_xxxxxxxxxxxxxxxxxxxxxxxxxxxx
NEXT_PUBLIC_APP_URL=https://yourdomain.com
```

---

## Utility Functions

### lib/paystack.ts

```typescript
const PAYSTACK_BASE_URL = 'https://api.paystack.co';

function getHeaders() {
  return {
    Authorization: `Bearer ${process.env.PAYSTACK_SECRET_KEY}`,
    'Content-Type': 'application/json',
  };
}

// Convert Naira to Kobo
export function toKobo(amount: number): number {
  return Math.round(amount * 100);
}

// Convert Kobo to Naira
export function toNaira(amount: number): number {
  return amount / 100;
}

// Generate unique reference
export function generateReference(prefix = 'TXN'): string {
  const timestamp = Date.now().toString(36);
  const random = Math.random().toString(36).substring(2, 8);
  return `${prefix}_${timestamp}_${random}`.toUpperCase();
}

// Initialize transaction
// NOTE: Omit 'reference' to let Paystack generate one (recommended).
// The returned reference MUST be stored in your database.
export async function initializeTransaction(params: {
  email: string;
  amount: number; // in kobo
  reference?: string; // Optional: only provide for retry scenarios
  callback_url?: string;
  metadata?: Record<string, unknown>;
  plan?: string;
  channels?: string[];
}) {
  const response = await fetch(`${PAYSTACK_BASE_URL}/transaction/initialize`, {
    method: 'POST',
    headers: getHeaders(),
    body: JSON.stringify(params), // Pass params as-is; Paystack generates reference if not provided
  });

  const result = await response.json();

  if (!result.status) {
    throw new Error(result.message || 'Failed to initialize transaction');
  }

  // IMPORTANT: result.data.reference is the authoritative reference
  // Store this in your database — it will appear in webhooks
  return result.data as {
    authorization_url: string;
    access_code: string;
    reference: string; // ← Store this in your database
  };
}

// Verify transaction
export async function verifyTransaction(reference: string) {
  const response = await fetch(
    `${PAYSTACK_BASE_URL}/transaction/verify/${encodeURIComponent(reference)}`,
    { headers: getHeaders() }
  );

  const result = await response.json();

  if (!result.status) {
    throw new Error(result.message || 'Failed to verify transaction');
  }

  return result.data;
}

// Verify webhook signature
export function verifyWebhookSignature(
  payload: string,
  signature: string
): boolean {
  const crypto = require('crypto');

  const hash = crypto
    .createHmac('sha512', process.env.PAYSTACK_SECRET_KEY!)
    .update(payload)
    .digest('hex');

  return hash.toLowerCase() === signature.toLowerCase();
}

// Create plan
export async function createPlan(params: {
  name: string;
  amount: number;
  interval: 'daily' | 'weekly' | 'monthly' | 'quarterly' | 'biannually' | 'annually';
  description?: string;
}) {
  const response = await fetch(`${PAYSTACK_BASE_URL}/plan`, {
    method: 'POST',
    headers: getHeaders(),
    body: JSON.stringify(params),
  });

  const result = await response.json();

  if (!result.status) {
    throw new Error(result.message || 'Failed to create plan');
  }

  return result.data;
}

// Fetch subscription
export async function getSubscription(code: string) {
  const response = await fetch(
    `${PAYSTACK_BASE_URL}/subscription/${encodeURIComponent(code)}`,
    { headers: getHeaders() }
  );

  const result = await response.json();

  if (!result.status) {
    throw new Error(result.message || 'Failed to fetch subscription');
  }

  return result.data;
}

// Disable subscription
export async function disableSubscription(code: string, token: string) {
  const response = await fetch(`${PAYSTACK_BASE_URL}/subscription/disable`, {
    method: 'POST',
    headers: getHeaders(),
    body: JSON.stringify({ code, token }),
  });

  const result = await response.json();

  if (!result.status) {
    throw new Error(result.message || 'Failed to disable subscription');
  }

  return result;
}
```

---

## API Routes

### app/api/paystack/initialize/route.ts

```typescript
import { NextRequest, NextResponse } from 'next/server';
import { initializeTransaction, toKobo } from '@/lib/paystack';

export async function POST(request: NextRequest) {
  try {
    const body = await request.json();

    const { email, amount, metadata, plan } = body;

    // Validation
    if (!email || !amount) {
      return NextResponse.json(
        { error: 'Email and amount are required' },
        { status: 400 }
      );
    }

    if (amount < 1) {
      return NextResponse.json(
        { error: 'Minimum amount is ₦1' },
        { status: 400 }
      );
    }

    // STEP 1: Call Paystack FIRST — let Paystack generate the reference
    // Do NOT provide a reference parameter; Paystack will generate one
    const transaction = await initializeTransaction({
      email,
      amount: toKobo(amount),
      callback_url: `${process.env.NEXT_PUBLIC_APP_URL}/payment/callback`,
      metadata,
      plan,
      // Note: reference is intentionally omitted — Paystack generates it
    });

    // STEP 2: Store order with Paystack's reference AFTER successful initialization
    // CRITICAL: Use transaction.reference (from Paystack), not a locally-generated one
    // This reference will appear in webhooks and must match your database
    // await db.orders.create({
    //   data: {
    //     reference: transaction.reference,  // ← Paystack's reference
    //     email,
    //     amount: toKobo(amount),
    //     status: 'pending',
    //     metadata,
    //   },
    // });

    return NextResponse.json({
      success: true,
      data: {
        authorization_url: transaction.authorization_url,
        access_code: transaction.access_code,
        reference: transaction.reference,
      },
    });
  } catch (error) {
    console.error('Initialize error:', error);
    return NextResponse.json(
      { error: error instanceof Error ? error.message : 'Failed to initialize' },
      { status: 500 }
    );
  }
}
```

### app/api/paystack/verify/route.ts

```typescript
import { NextRequest, NextResponse } from 'next/server';
import { verifyTransaction } from '@/lib/paystack';

export async function GET(request: NextRequest) {
  try {
    const { searchParams } = new URL(request.url);
    const reference = searchParams.get('reference');

    if (!reference) {
      return NextResponse.json(
        { error: 'Reference is required' },
        { status: 400 }
      );
    }

    // Get expected amount from your database
    // const order = await db.orders.findUnique({
    //   where: { reference },
    // });
    //
    // if (!order) {
    //   return NextResponse.json(
    //     { error: 'Order not found' },
    //     { status: 404 }
    //   );
    // }

    const transaction = await verifyTransaction(reference);

    // Verify status
    if (transaction.status !== 'success') {
      return NextResponse.json({
        success: false,
        message: `Payment ${transaction.status}`,
        data: transaction,
      });
    }

    // Verify amount (CRITICAL)
    // if (transaction.amount !== order.amount) {
    //   return NextResponse.json({
    //     success: false,
    //     message: 'Amount mismatch',
    //   });
    // }

    // Update order status
    // await db.orders.update({
    //   where: { reference },
    //   data: {
    //     status: 'paid',
    //     paidAt: new Date(),
    //     transactionId: transaction.id,
    //   },
    // });

    return NextResponse.json({
      success: true,
      message: 'Payment verified',
      data: {
        reference: transaction.reference,
        amount: transaction.amount,
        status: transaction.status,
        customer: transaction.customer,
        paid_at: transaction.paid_at,
      },
    });
  } catch (error) {
    console.error('Verify error:', error);
    return NextResponse.json(
      { error: error instanceof Error ? error.message : 'Verification failed' },
      { status: 500 }
    );
  }
}
```

### app/api/paystack/webhook/route.ts

```typescript
import { NextRequest, NextResponse } from 'next/server';
import { verifyWebhookSignature } from '@/lib/paystack';

export async function POST(request: NextRequest) {
  try {
    // Get raw body for signature verification
    const rawBody = await request.text();
    const signature = request.headers.get('x-paystack-signature') || '';

    // Verify signature (CRITICAL)
    if (!verifyWebhookSignature(rawBody, signature)) {
      console.error('Invalid webhook signature');
      return NextResponse.json(
        { error: 'Invalid signature' },
        { status: 401 }
      );
    }

    const event = JSON.parse(rawBody);

    console.log('Webhook event:', event.event);

    // Handle events
    switch (event.event) {
      case 'charge.success':
        await handleChargeSuccess(event.data);
        break;

      case 'charge.failed':
        await handleChargeFailed(event.data);
        break;

      case 'subscription.create':
        await handleSubscriptionCreate(event.data);
        break;

      case 'subscription.disable':
        await handleSubscriptionDisable(event.data);
        break;

      case 'invoice.payment_failed':
        await handleInvoicePaymentFailed(event.data);
        break;

      default:
        console.log('Unhandled event:', event.event);
    }

    // Must return 200 quickly
    return NextResponse.json({ received: true });
  } catch (error) {
    console.error('Webhook error:', error);
    // Still return 200 to prevent retries for our errors
    return NextResponse.json({ received: true });
  }
}

// Event handlers
async function handleChargeSuccess(data: Record<string, unknown>) {
  const reference = data.reference as string;
  const amount = data.amount as number;

  console.log(`Payment successful: ${reference}, Amount: ${amount / 100} NGN`);

  // Update your database
  // await db.orders.update({
  //   where: { reference },
  //   data: { status: 'paid', paidAt: new Date() },
  // });

  // Send confirmation email, fulfill order, etc.
}

async function handleChargeFailed(data: Record<string, unknown>) {
  const reference = data.reference as string;
  const reason = data.gateway_response as string;

  console.log(`Payment failed: ${reference}, Reason: ${reason}`);

  // Update order status, notify user
}

async function handleSubscriptionCreate(data: Record<string, unknown>) {
  const subscriptionCode = data.subscription_code as string;
  const customerEmail = (data.customer as { email: string }).email;
  const planCode = (data.plan as { plan_code: string }).plan_code;

  console.log(`Subscription created: ${subscriptionCode} for ${customerEmail}`);

  // Activate user subscription
  // await db.users.update({
  //   where: { email: customerEmail },
  //   data: {
  //     subscriptionCode,
  //     subscriptionStatus: 'active',
  //     planCode,
  //   },
  // });
}

async function handleSubscriptionDisable(data: Record<string, unknown>) {
  const subscriptionCode = data.subscription_code as string;

  console.log(`Subscription cancelled: ${subscriptionCode}`);

  // Revoke access
  // await db.users.update({
  //   where: { subscriptionCode },
  //   data: { subscriptionStatus: 'cancelled' },
  // });
}

async function handleInvoicePaymentFailed(data: Record<string, unknown>) {
  const customerEmail = (data.customer as { email: string }).email;

  console.log(`Invoice payment failed for ${customerEmail}`);

  // Notify user to update payment method
}
```

---

## Client Components

### components/PaystackButton.tsx

```typescript
'use client';

import { useState, useCallback } from 'react';
import PaystackPop from '@paystack/inline-js';

interface PaystackButtonProps {
  email: string;
  amount: number; // in Naira
  onSuccess?: (reference: string) => void;
  onCancel?: () => void;
  metadata?: Record<string, unknown>;
  plan?: string;
  className?: string;
  children?: React.ReactNode;
  disabled?: boolean;
}

export function PaystackButton({
  email,
  amount,
  onSuccess,
  onCancel,
  metadata,
  plan,
  className = '',
  children,
  disabled = false,
}: PaystackButtonProps) {
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const handlePayment = useCallback(async () => {
    setLoading(true);
    setError(null);

    try {
      // Initialize on backend
      const response = await fetch('/api/paystack/initialize', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          email,
          amount,
          metadata,
          plan,
        }),
      });

      const result = await response.json();

      if (!result.success) {
        throw new Error(result.error || 'Failed to initialize payment');
      }

      const { access_code, reference } = result.data;

      // Open Paystack popup
      const popup = new PaystackPop();
      popup.newTransaction({
        key: process.env.NEXT_PUBLIC_PAYSTACK_PUBLIC_KEY!,
        accessCode: access_code,
        onSuccess: async (transaction) => {
          console.log('Payment successful:', transaction);

          // Verify on backend
          const verifyResponse = await fetch(
            `/api/paystack/verify?reference=${reference}`
          );
          const verifyResult = await verifyResponse.json();

          if (verifyResult.success) {
            onSuccess?.(reference);
          } else {
            setError('Payment verification failed');
          }
        },
        onCancel: () => {
          console.log('Payment cancelled');
          onCancel?.();
        },
      });
    } catch (err) {
      console.error('Payment error:', err);
      setError(err instanceof Error ? err.message : 'Payment failed');
    } finally {
      setLoading(false);
    }
  }, [email, amount, metadata, plan, onSuccess, onCancel]);

  return (
    <div>
      <button
        onClick={handlePayment}
        disabled={disabled || loading}
        className={`${className} ${loading ? 'opacity-50 cursor-not-allowed' : ''}`}
      >
        {loading ? 'Processing...' : children || `Pay ₦${amount.toLocaleString()}`}
      </button>
      {error && <p className="text-red-500 text-sm mt-2">{error}</p>}
    </div>
  );
}
```

### Usage in a Page

```typescript
// app/payment/page.tsx
'use client';

import { useRouter } from 'next/navigation';
import { PaystackButton } from '@/components/PaystackButton';

export default function PaymentPage() {
  const router = useRouter();

  return (
    <div className="max-w-md mx-auto p-6">
      <h1 className="text-2xl font-bold mb-6">Complete Payment</h1>

      <div className="bg-white shadow rounded-lg p-6">
        <div className="mb-4">
          <p className="text-gray-600">Product: Premium Plan</p>
          <p className="text-2xl font-bold">₦5,000</p>
        </div>

        <PaystackButton
          email="customer@example.com"
          amount={5000}
          metadata={{ product: 'premium_plan', user_id: 'usr_123' }}
          onSuccess={(reference) => {
            router.push(`/payment/success?reference=${reference}`);
          }}
          onCancel={() => {
            alert('Payment was cancelled');
          }}
          className="w-full bg-green-600 hover:bg-green-700 text-white font-bold py-3 px-4 rounded-lg"
        >
          Pay ₦5,000
        </PaystackButton>
      </div>
    </div>
  );
}
```

---

## Webhook Handler

### app/payment/callback/page.tsx

Handle redirect after payment:

```typescript
// app/payment/callback/page.tsx
import { redirect } from 'next/navigation';
import { verifyTransaction } from '@/lib/paystack';

interface Props {
  searchParams: { reference?: string; trxref?: string };
}

export default async function PaymentCallbackPage({ searchParams }: Props) {
  const reference = searchParams.reference || searchParams.trxref;

  if (!reference) {
    redirect('/payment/error?reason=missing_reference');
  }

  try {
    const transaction = await verifyTransaction(reference);

    if (transaction.status === 'success') {
      // Update database, fulfill order
      redirect(`/payment/success?reference=${reference}`);
    } else {
      redirect(`/payment/failed?reason=${transaction.status}`);
    }
  } catch (error) {
    console.error('Callback error:', error);
    redirect('/payment/error?reason=verification_failed');
  }
}
```

### app/payment/success/page.tsx

```typescript
interface Props {
  searchParams: { reference?: string };
}

export default function PaymentSuccessPage({ searchParams }: Props) {
  return (
    <div className="max-w-md mx-auto p-6 text-center">
      <div className="text-green-500 text-6xl mb-4">✓</div>
      <h1 className="text-2xl font-bold mb-2">Payment Successful!</h1>
      <p className="text-gray-600 mb-4">
        Your payment has been processed successfully.
      </p>
      {searchParams.reference && (
        <p className="text-sm text-gray-500">
          Reference: {searchParams.reference}
        </p>
      )}
      <a
        href="/"
        className="inline-block mt-6 bg-green-600 text-white px-6 py-2 rounded-lg"
      >
        Continue
      </a>
    </div>
  );
}
```

---

## Complete Example

### Subscription Checkout Flow

```typescript
// app/subscribe/page.tsx
'use client';

import { useState } from 'react';
import { useRouter } from 'next/navigation';
import { PaystackButton } from '@/components/PaystackButton';

const PLANS = [
  { code: 'PLN_basic', name: 'Basic', price: 2999, features: ['5 Projects', 'Basic Support'] },
  { code: 'PLN_pro', name: 'Pro', price: 9999, features: ['Unlimited Projects', 'Priority Support', 'API Access'] },
];

export default function SubscribePage() {
  const router = useRouter();
  const [selectedPlan, setSelectedPlan] = useState(PLANS[0]);
  const [email, setEmail] = useState('');

  return (
    <div className="max-w-4xl mx-auto p-6">
      <h1 className="text-3xl font-bold mb-8">Choose Your Plan</h1>

      <div className="grid md:grid-cols-2 gap-6 mb-8">
        {PLANS.map((plan) => (
          <div
            key={plan.code}
            onClick={() => setSelectedPlan(plan)}
            className={`border-2 rounded-lg p-6 cursor-pointer transition ${
              selectedPlan.code === plan.code
                ? 'border-green-500 bg-green-50'
                : 'border-gray-200 hover:border-gray-300'
            }`}
          >
            <h3 className="text-xl font-bold">{plan.name}</h3>
            <p className="text-3xl font-bold my-4">
              ₦{plan.price.toLocaleString()}
              <span className="text-sm font-normal text-gray-500">/month</span>
            </p>
            <ul className="space-y-2">
              {plan.features.map((feature) => (
                <li key={feature} className="text-gray-600">✓ {feature}</li>
              ))}
            </ul>
          </div>
        ))}
      </div>

      <div className="max-w-md">
        <label className="block mb-2 font-medium">Email</label>
        <input
          type="email"
          value={email}
          onChange={(e) => setEmail(e.target.value)}
          className="w-full border rounded-lg px-4 py-2 mb-4"
          placeholder="you@example.com"
        />

        <PaystackButton
          email={email}
          amount={selectedPlan.price}
          plan={selectedPlan.code}
          metadata={{ plan_name: selectedPlan.name }}
          onSuccess={(reference) => {
            router.push(`/subscribe/success?reference=${reference}`);
          }}
          disabled={!email}
          className="w-full bg-green-600 hover:bg-green-700 text-white font-bold py-3 px-4 rounded-lg disabled:opacity-50"
        >
          Subscribe to {selectedPlan.name} - ₦{selectedPlan.price.toLocaleString()}/month
        </PaystackButton>
      </div>
    </div>
  );
}
```
