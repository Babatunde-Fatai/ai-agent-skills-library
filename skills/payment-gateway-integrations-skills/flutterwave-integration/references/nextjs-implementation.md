> **When to read:** When implementing Flutterwave routes with Next.js App Router.
> **What problem it solves:** Shows file placement, raw-body webhook handling, and complete patterns for Next.js.
> **When to skip:** Not needed for non-Next.js projects.
> **Prerequisites:** Read AGENT_EXECUTION_SPEC.md first.

# Next.js Implementation - Complete Guide

Complete Flutterwave integration for Next.js 13+ App Router with TypeScript.

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
│   │   └── flutterwave/
│   │       ├── initialize/
│   │       │   └── route.ts       # POST - Initialize payment
│   │       ├── verify/
│   │       │   └── route.ts       # GET - Verify payment
│   │       └── webhook/
│   │           └── route.ts       # POST - Handle webhooks
│   ├── payment/
│   │   ├── page.tsx               # Payment page
│   │   ├── success/
│   │   │   └── page.tsx           # Success page
│   │   └── callback/
│   │       └── page.tsx           # Callback handler
│   └── layout.tsx
├── components/
│   └── FlutterwaveButton.tsx      # Payment button component
├── lib/
│   └── flutterwave.ts             # Utility functions
├── types/
│   └── flutterwave.ts             # TypeScript types
└── .env.local
```

---

## Environment Setup

### .env.local

```bash
# Flutterwave Keys
FLW_SECRET_KEY=FLWSECK_TEST-xxxxxxxxxxxxxxxxxxxx
FLW_SECRET_HASH=your_webhook_secret_hash
NEXT_PUBLIC_FLW_PUBLIC_KEY=FLWPUBK_TEST-xxxxxxxxxxxxxxxxxxxx

# App URL (for callbacks)
NEXT_PUBLIC_APP_URL=http://localhost:3000
```

### .env.production

```bash
FLW_SECRET_KEY=FLWSECK-xxxxxxxxxxxxxxxxxxxx
FLW_SECRET_HASH=your_webhook_secret_hash
NEXT_PUBLIC_FLW_PUBLIC_KEY=FLWPUBK-xxxxxxxxxxxxxxxxxxxx
NEXT_PUBLIC_APP_URL=https://yourdomain.com
```

---

## Utility Functions

### lib/flutterwave.ts

```typescript
import crypto from 'crypto';

const FLW_BASE_URL = 'https://api.flutterwave.com/v3';

function getHeaders() {
  return {
    Authorization: `Bearer ${process.env.FLW_SECRET_KEY}`,
    'Content-Type': 'application/json',
  };
}

// Convert to smallest currency unit
export function toSmallestUnit(amount: number): number {
  return Math.round(amount * 100);
}

// Convert from smallest unit
export function fromSmallestUnit(amount: number): number {
  return amount / 100;
}

// Generate unique tx_ref
export function generateTxRef(prefix = 'FLW'): string {
  const timestamp = Date.now().toString(36);
  const random = crypto.randomBytes(4).toString('hex');
  return `${prefix}_${timestamp}_${random}`.toUpperCase();
}

// Initialize payment
export async function initializePayment(params: {
  tx_ref: string;
  amount: number;
  currency: string;
  redirect_url: string;
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
  payment_options?: string;
  meta?: Record<string, unknown>;
}) {
  const response = await fetch(`${FLW_BASE_URL}/payments`, {
    method: 'POST',
    headers: getHeaders(),
    body: JSON.stringify(params),
  });

  const result = await response.json();

  if (result.status !== 'success') {
    throw new Error(result.message || 'Failed to initialize payment');
  }

  return result.data as { link: string };
}

// Verify transaction
export async function verifyTransaction(transactionId: number) {
  const response = await fetch(
    `${FLW_BASE_URL}/transactions/${transactionId}/verify`,
    { headers: getHeaders() }
  );

  const result = await response.json();

  if (result.status !== 'success') {
    throw new Error(result.message || 'Failed to verify transaction');
  }

  return result.data;
}

// Verify by tx_ref
export async function verifyByTxRef(txRef: string) {
  const response = await fetch(
    `${FLW_BASE_URL}/transactions/verify_by_reference?tx_ref=${encodeURIComponent(txRef)}`,
    { headers: getHeaders() }
  );

  const result = await response.json();

  if (result.status !== 'success') {
    throw new Error(result.message || 'Failed to verify transaction');
  }

  return result.data;
}

// Verify webhook signature (simple verif-hash method)
export function verifyWebhookSignature(signature: string | null): boolean {
  const secretHash = process.env.FLW_SECRET_HASH;
  if (!signature || !secretHash) {
    return false;
  }
  return signature === secretHash;
}

// Verify webhook signature (HMAC method)
export function verifyWebhookHMAC(
  payload: string,
  signature: string
): boolean {
  const secretHash = process.env.FLW_SECRET_HASH!;

  const hash = crypto
    .createHmac('sha256', secretHash)
    .update(payload)
    .digest('base64');

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

---

## API Routes

### app/api/flutterwave/initialize/route.ts

```typescript
import { NextRequest, NextResponse } from 'next/server';
import { initializePayment, toSmallestUnit, generateTxRef } from '@/lib/flutterwave';

export async function POST(request: NextRequest) {
  try {
    const body = await request.json();
    const { email, amount, currency = 'NGN', name, metadata } = body;

    // Validation
    if (!email || !amount) {
      return NextResponse.json(
        { error: 'Email and amount are required' },
        { status: 400 }
      );
    }

    if (amount < 1) {
      return NextResponse.json(
        { error: 'Minimum amount is 1' },
        { status: 400 }
      );
    }

    // STEP 1: Generate unique tx_ref
    const txRef = generateTxRef('PAY');
    const amountInSmallestUnit = toSmallestUnit(amount);

    // STEP 2: Store order in database FIRST
    // await db.orders.create({
    //   data: {
    //     txRef,
    //     email,
    //     amount: amountInSmallestUnit,
    //     currency,
    //     status: 'pending',
    //     metadata,
    //   },
    // });

    // STEP 3: Initialize payment with Flutterwave
    const payment = await initializePayment({
      tx_ref: txRef,
      amount: amountInSmallestUnit,
      currency,
      redirect_url: `${process.env.NEXT_PUBLIC_APP_URL}/payment/callback`,
      customer: {
        email,
        name,
      },
      customizations: {
        title: 'My Store',
        description: 'Payment for your order',
      },
      meta: metadata,
    });

    return NextResponse.json({
      success: true,
      data: {
        link: payment.link,
        txRef,
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

### app/api/flutterwave/verify/route.ts

```typescript
import { NextRequest, NextResponse } from 'next/server';
import { verifyTransaction, verifyByTxRef } from '@/lib/flutterwave';

export async function GET(request: NextRequest) {
  try {
    const { searchParams } = new URL(request.url);
    const transactionId = searchParams.get('transaction_id');
    const txRef = searchParams.get('tx_ref');

    if (!transactionId && !txRef) {
      return NextResponse.json(
        { error: 'transaction_id or tx_ref is required' },
        { status: 400 }
      );
    }

    // Verify with Flutterwave
    const transaction = transactionId
      ? await verifyTransaction(parseInt(transactionId))
      : await verifyByTxRef(txRef!);

    // Get expected amount from database
    // const order = await db.orders.findUnique({
    //   where: { txRef: transaction.tx_ref },
    // });
    //
    // if (!order) {
    //   return NextResponse.json(
    //     { error: 'Order not found' },
    //     { status: 404 }
    //   );
    // }

    // Verify status
    if (transaction.status !== 'successful') {
      return NextResponse.json({
        success: false,
        message: `Payment ${transaction.status}`,
        data: transaction,
      });
    }

    // CRITICAL: Verify amount matches
    // if (transaction.amount !== order.amount) {
    //   return NextResponse.json({
    //     success: false,
    //     message: 'Amount mismatch',
    //   });
    // }

    // Update order status
    // await db.orders.update({
    //   where: { txRef: transaction.tx_ref },
    //   data: {
    //     status: 'paid',
    //     paidAt: new Date(),
    //     transactionId: transaction.id,
    //     flwRef: transaction.flw_ref,
    //   },
    // });

    return NextResponse.json({
      success: true,
      message: 'Payment verified',
      data: {
        txRef: transaction.tx_ref,
        amount: transaction.amount,
        currency: transaction.currency,
        status: transaction.status,
        customer: transaction.customer,
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

### app/api/flutterwave/webhook/route.ts

```typescript
import { NextRequest, NextResponse } from 'next/server';
import { verifyWebhookSignature } from '@/lib/flutterwave';

export async function POST(request: NextRequest) {
  try {
    // Get raw body for signature verification
    const rawBody = await request.text();
    const signature = request.headers.get('verif-hash');

    // Verify signature (CRITICAL)
    if (!verifyWebhookSignature(signature)) {
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
      case 'charge.completed':
        await handleChargeCompleted(event.data);
        break;

      case 'charge.failed':
        await handleChargeFailed(event.data);
        break;

      case 'transfer.completed':
        await handleTransferCompleted(event.data);
        break;

      case 'transfer.failed':
        await handleTransferFailed(event.data);
        break;

      default:
        console.log('Unhandled event:', event.event);
    }

    // Must return 200 quickly
    return NextResponse.json({ received: true });
  } catch (error) {
    console.error('Webhook error:', error);
    // Return 200 to prevent retries
    return NextResponse.json({ received: true });
  }
}

// Event handlers
async function handleChargeCompleted(data: Record<string, unknown>) {
  const txRef = data.tx_ref as string;
  const amount = data.amount as number;
  const status = data.status as string;

  if (status !== 'successful') {
    console.log(`Charge ${txRef} status is ${status}, not processing`);
    return;
  }

  console.log(`Payment successful: ${txRef}, Amount: ${amount}`);

  // Get order from database
  // const order = await db.orders.findUnique({ where: { txRef } });
  // if (!order || order.status === 'paid') return;

  // CRITICAL: Verify amount matches
  // if (order.amount !== amount) {
  //   console.error(`Amount mismatch for ${txRef}`);
  //   return;
  // }

  // Update order
  // await db.orders.update({
  //   where: { txRef },
  //   data: { status: 'paid', paidAt: new Date() },
  // });
}

async function handleChargeFailed(data: Record<string, unknown>) {
  const txRef = data.tx_ref as string;
  const reason = data.processor_response as string;

  console.log(`Payment failed: ${txRef}, Reason: ${reason}`);

  // Update order status
  // await db.orders.update({
  //   where: { txRef },
  //   data: { status: 'failed', failureReason: reason },
  // });
}

async function handleTransferCompleted(data: Record<string, unknown>) {
  const reference = data.reference as string;
  console.log(`Transfer completed: ${reference}`);
}

async function handleTransferFailed(data: Record<string, unknown>) {
  const reference = data.reference as string;
  const reason = data.complete_message as string;
  console.log(`Transfer failed: ${reference} - ${reason}`);
}
```

---

## Client Components

### components/FlutterwaveButton.tsx

```typescript
'use client';

import { useState, useCallback } from 'react';

interface FlutterwaveButtonProps {
  email: string;
  amount: number;  // In display currency (not smallest unit)
  currency?: string;
  name?: string;
  onSuccess?: (txRef: string) => void;
  onCancel?: () => void;
  metadata?: Record<string, unknown>;
  className?: string;
  children?: React.ReactNode;
  disabled?: boolean;
}

export function FlutterwaveButton({
  email,
  amount,
  currency = 'NGN',
  name,
  onSuccess,
  onCancel,
  metadata,
  className = '',
  children,
  disabled = false,
}: FlutterwaveButtonProps) {
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const handlePayment = useCallback(async () => {
    setLoading(true);
    setError(null);

    try {
      // Initialize on backend
      const response = await fetch('/api/flutterwave/initialize', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          email,
          amount,
          currency,
          name,
          metadata,
        }),
      });

      const result = await response.json();

      if (!result.success) {
        throw new Error(result.error || 'Failed to initialize payment');
      }

      const { link, txRef } = result.data;

      // Store txRef for later verification
      sessionStorage.setItem('flw_tx_ref', txRef);

      // Redirect to Flutterwave
      window.location.href = link;

    } catch (err) {
      console.error('Payment error:', err);
      setError(err instanceof Error ? err.message : 'Payment failed');
      onCancel?.();
    } finally {
      setLoading(false);
    }
  }, [email, amount, currency, name, metadata, onCancel]);

  const currencySymbols: Record<string, string> = {
    NGN: '₦',
    GHS: 'GH₵',
    KES: 'KSh',
    ZAR: 'R',
    USD: '$',
  };

  const symbol = currencySymbols[currency] || currency;

  return (
    <div>
      <button
        onClick={handlePayment}
        disabled={disabled || loading}
        className={`${className} ${loading ? 'opacity-50 cursor-not-allowed' : ''}`}
      >
        {loading ? 'Processing...' : children || `Pay ${symbol}${amount.toLocaleString()}`}
      </button>
      {error && <p className="text-red-500 text-sm mt-2">{error}</p>}
    </div>
  );
}
```

### Using Flutterwave Inline (Optional)

For popup instead of redirect:

```typescript
'use client';

import { useCallback } from 'react';

declare global {
  interface Window {
    FlutterwaveCheckout: (config: any) => void;
  }
}

export function FlutterwaveInlineButton({
  email,
  amount,
  currency = 'NGN',
  txRef,
  onSuccess,
  onClose,
}: {
  email: string;
  amount: number;
  currency?: string;
  txRef: string;
  onSuccess: (response: any) => void;
  onClose: () => void;
}) {
  const handlePayment = useCallback(() => {
    window.FlutterwaveCheckout({
      public_key: process.env.NEXT_PUBLIC_FLW_PUBLIC_KEY,
      tx_ref: txRef,
      amount: amount,
      currency: currency,
      payment_options: 'card,mobilemoney,ussd,banktransfer',
      customer: {
        email: email,
      },
      customizations: {
        title: 'My Store',
        description: 'Payment for your order',
      },
      callback: (response: any) => {
        console.log('Payment response:', response);
        onSuccess(response);
      },
      onclose: () => {
        console.log('Payment closed');
        onClose();
      },
    });
  }, [email, amount, currency, txRef, onSuccess, onClose]);

  return (
    <>
      <script src="https://checkout.flutterwave.com/v3.js" async />
      <button onClick={handlePayment}>
        Pay {currency} {amount}
      </button>
    </>
  );
}
```

---

## Webhook Handler

### app/payment/callback/page.tsx

Handle redirect after payment:

```typescript
import { redirect } from 'next/navigation';
import { verifyTransaction } from '@/lib/flutterwave';

interface Props {
  searchParams: {
    status?: string;
    tx_ref?: string;
    transaction_id?: string;
  };
}

export default async function PaymentCallbackPage({ searchParams }: Props) {
  const { status, tx_ref, transaction_id } = searchParams;

  if (!tx_ref || !transaction_id) {
    redirect('/payment/error?reason=missing_params');
  }

  try {
    // CRITICAL: Don't trust status param, verify with Flutterwave
    const transaction = await verifyTransaction(parseInt(transaction_id));

    if (transaction.status === 'successful') {
      // Verify amount matches your database
      // const order = await db.orders.findUnique({ where: { txRef: tx_ref } });
      // if (transaction.amount !== order.amount) {
      //   redirect('/payment/error?reason=amount_mismatch');
      // }

      // Update database
      // await db.orders.update({
      //   where: { txRef: tx_ref },
      //   data: { status: 'paid', paidAt: new Date() },
      // });

      redirect(`/payment/success?tx_ref=${tx_ref}`);
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
  searchParams: { tx_ref?: string };
}

export default function PaymentSuccessPage({ searchParams }: Props) {
  return (
    <div className="max-w-md mx-auto p-6 text-center">
      <div className="text-green-500 text-6xl mb-4">✓</div>
      <h1 className="text-2xl font-bold mb-2">Payment Successful!</h1>
      <p className="text-gray-600 mb-4">
        Your payment has been processed successfully.
      </p>
      {searchParams.tx_ref && (
        <p className="text-sm text-gray-500">
          Reference: {searchParams.tx_ref}
        </p>
      )}
      <a
        href="/"
        className="inline-block mt-6 bg-orange-500 text-white px-6 py-2 rounded-lg"
      >
        Continue
      </a>
    </div>
  );
}
```

---

## Complete Example

### Checkout Flow

```typescript
// app/checkout/page.tsx
'use client';

import { useState } from 'react';
import { useRouter } from 'next/navigation';
import { FlutterwaveButton } from '@/components/FlutterwaveButton';

export default function CheckoutPage() {
  const router = useRouter();
  const [email, setEmail] = useState('');

  // Example cart data
  const cart = {
    items: ['Product A', 'Product B'],
    total: 15000,  // 15,000 NGN
    currency: 'NGN',
  };

  return (
    <div className="max-w-md mx-auto p-6">
      <h1 className="text-2xl font-bold mb-6">Checkout</h1>

      <div className="bg-white shadow rounded-lg p-6 mb-6">
        <h2 className="font-bold mb-4">Order Summary</h2>
        <ul className="space-y-2 mb-4">
          {cart.items.map((item, i) => (
            <li key={i} className="text-gray-600">{item}</li>
          ))}
        </ul>
        <div className="border-t pt-4">
          <div className="flex justify-between font-bold">
            <span>Total</span>
            <span>₦{cart.total.toLocaleString()}</span>
          </div>
        </div>
      </div>

      <div className="mb-6">
        <label className="block mb-2 font-medium">Email</label>
        <input
          type="email"
          value={email}
          onChange={(e) => setEmail(e.target.value)}
          className="w-full border rounded-lg px-4 py-2"
          placeholder="you@example.com"
          required
        />
      </div>

      <FlutterwaveButton
        email={email}
        amount={cart.total}
        currency={cart.currency}
        metadata={{
          items: cart.items,
        }}
        onSuccess={(txRef) => {
          router.push(`/payment/success?tx_ref=${txRef}`);
        }}
        onCancel={() => {
          alert('Payment was cancelled');
        }}
        disabled={!email}
        className="w-full bg-orange-500 hover:bg-orange-600 text-white font-bold py-3 px-4 rounded-lg disabled:opacity-50"
      >
        Pay ₦{cart.total.toLocaleString()}
      </FlutterwaveButton>
    </div>
  );
}
```
