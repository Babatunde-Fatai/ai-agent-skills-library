> **When to read:** When implementing backend verification and webhook handling in Express.js.
> **What problem it solves:** Maps Agent Execution Spec roles to Express project structure.
> **When to skip:** If you are not using Express/Node or only work on frontend UI.
> **Prerequisites:** Read AGENT_EXECUTION_SPEC.md first.

# Express.js Implementation - Complete Guide

Complete Paystack integration for Express.js with TypeScript.

## Table of Contents

1. [Project Structure](#project-structure)
2. [Setup](#setup)
3. [Services](#services)
4. [Controllers](#controllers)
5. [Routes](#routes)
6. [Middleware](#middleware)
7. [Complete Example](#complete-example)

---

## Project Structure

```
your-express-app/
├── src/
│   ├── config/
│   │   └── paystack.ts            # Configuration
│   ├── controllers/
│   │   └── paystack.controller.ts # Route handlers
│   ├── middleware/
│   │   └── paystack.middleware.ts # Webhook verification
│   ├── routes/
│   │   └── paystack.routes.ts     # Route definitions
│   ├── services/
│   │   └── paystack.service.ts    # API calls
│   ├── types/
│   │   └── paystack.types.ts      # TypeScript types
│   └── app.ts                     # Express app
├── .env
└── package.json
```

---

## Setup

### Install Dependencies

```bash
npm install express dotenv
npm install -D typescript @types/express @types/node
```

### .env

```bash
PORT=3000
PAYSTACK_SECRET_KEY=sk_test_xxxxxxxxxxxxxxxxxxxx
PAYSTACK_PUBLIC_KEY=pk_test_xxxxxxxxxxxxxxxxxxxx
FRONTEND_URL=http://localhost:3001
```

### src/config/paystack.ts

```typescript
export const PAYSTACK_CONFIG = {
  baseUrl: 'https://api.paystack.co',
  secretKey: process.env.PAYSTACK_SECRET_KEY!,
  publicKey: process.env.PAYSTACK_PUBLIC_KEY!,
};

export const getPaystackHeaders = () => ({
  Authorization: `Bearer ${PAYSTACK_CONFIG.secretKey}`,
  'Content-Type': 'application/json',
});
```

---

## Services

### src/services/paystack.service.ts

```typescript
import crypto from 'crypto';
import { PAYSTACK_CONFIG, getPaystackHeaders } from '../config/paystack';

const BASE_URL = PAYSTACK_CONFIG.baseUrl;

// Utility functions
export function toKobo(amount: number): number {
  return Math.round(amount * 100);
}

export function toNaira(amount: number): number {
  return amount / 100;
}

export function generateReference(prefix = 'TXN'): string {
  const timestamp = Date.now().toString(36);
  const random = crypto.randomBytes(4).toString('hex');
  return `${prefix}_${timestamp}_${random}`.toUpperCase();
}

// Initialize transaction
// NOTE: Omit 'reference' to let Paystack generate one (recommended).
// The returned reference MUST be stored in your database.
export async function initializeTransaction(params: {
  email: string;
  amount: number;
  reference?: string; // Optional: only provide for retry scenarios
  callback_url?: string;
  metadata?: Record<string, unknown>;
  plan?: string;
  channels?: string[];
}) {
  const response = await fetch(`${BASE_URL}/transaction/initialize`, {
    method: 'POST',
    headers: getPaystackHeaders(),
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
    `${BASE_URL}/transaction/verify/${encodeURIComponent(reference)}`,
    { headers: getPaystackHeaders() }
  );

  const result = await response.json();

  if (!result.status) {
    throw new Error(result.message || 'Failed to verify transaction');
  }

  return result.data;
}

// Charge authorization (recurring)
export async function chargeAuthorization(params: {
  authorization_code: string;
  email: string;
  amount: number;
  reference?: string;
  metadata?: Record<string, unknown>;
}) {
  const response = await fetch(`${BASE_URL}/transaction/charge_authorization`, {
    method: 'POST',
    headers: getPaystackHeaders(),
    body: JSON.stringify({
      ...params,
      reference: params.reference || generateReference('CHG'),
    }),
  });

  const result = await response.json();

  if (!result.status) {
    throw new Error(result.message || 'Failed to charge authorization');
  }

  return result.data;
}

// Create plan
export async function createPlan(params: {
  name: string;
  amount: number;
  interval: 'daily' | 'weekly' | 'monthly' | 'quarterly' | 'biannually' | 'annually';
  description?: string;
}) {
  const response = await fetch(`${BASE_URL}/plan`, {
    method: 'POST',
    headers: getPaystackHeaders(),
    body: JSON.stringify(params),
  });

  const result = await response.json();

  if (!result.status) {
    throw new Error(result.message || 'Failed to create plan');
  }

  return result.data;
}

// Get subscription
export async function getSubscription(code: string) {
  const response = await fetch(
    `${BASE_URL}/subscription/${encodeURIComponent(code)}`,
    { headers: getPaystackHeaders() }
  );

  const result = await response.json();

  if (!result.status) {
    throw new Error(result.message || 'Failed to get subscription');
  }

  return result.data;
}

// Disable subscription
export async function disableSubscription(code: string, token: string) {
  const response = await fetch(`${BASE_URL}/subscription/disable`, {
    method: 'POST',
    headers: getPaystackHeaders(),
    body: JSON.stringify({ code, token }),
  });

  const result = await response.json();

  if (!result.status) {
    throw new Error(result.message || 'Failed to disable subscription');
  }

  return result;
}

// Verify webhook signature (timing-safe comparison)
export function verifyWebhookSignature(
  payload: string | Buffer,
  signature: string
): boolean {
  const payloadString = typeof payload === 'string' ? payload : payload.toString('utf8');

  const hash = crypto
    .createHmac('sha512', PAYSTACK_CONFIG.secretKey)
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
```

---

## Controllers

### src/controllers/paystack.controller.ts

```typescript
import { Request, Response } from 'express';
import * as PaystackService from '../services/paystack.service';

// Initialize payment
export async function initializePayment(req: Request, res: Response) {
  try {
    const { email, amount, metadata, plan } = req.body;

    // Validation
    if (!email || !amount) {
      return res.status(400).json({
        success: false,
        error: 'Email and amount are required',
      });
    }

    if (amount < 1) {
      return res.status(400).json({
        success: false,
        error: 'Minimum amount is ₦1',
      });
    }

    // STEP 1: Call Paystack FIRST — let Paystack generate the reference
    // Do NOT provide a reference parameter; Paystack will generate one
    const transaction = await PaystackService.initializeTransaction({
      email,
      amount: PaystackService.toKobo(amount),
      callback_url: `${process.env.FRONTEND_URL}/payment/callback`,
      metadata,
      plan,
      // Note: reference is intentionally omitted — Paystack generates it
    });

    // STEP 2: Store order with Paystack's reference AFTER successful initialization
    // CRITICAL: Use transaction.reference (from Paystack), not a locally-generated one
    // This reference will appear in webhooks and must match your database
    // TODO: Store in database
    // await db.orders.create({
    //   reference: transaction.reference,  // ← Paystack's reference
    //   email,
    //   amount: PaystackService.toKobo(amount),
    //   status: 'pending',
    // });

    return res.json({
      success: true,
      data: transaction,
    });
  } catch (error) {
    console.error('Initialize error:', error);
    return res.status(500).json({
      success: false,
      error: error instanceof Error ? error.message : 'Failed to initialize',
    });
  }
}

// Verify payment
export async function verifyPayment(req: Request, res: Response) {
  try {
    const { reference } = req.query;

    if (!reference || typeof reference !== 'string') {
      return res.status(400).json({
        success: false,
        error: 'Reference is required',
      });
    }

    // TODO: Get expected amount from database
    // const order = await db.orders.findUnique({ where: { reference } });
    // if (!order) {
    //   return res.status(404).json({ success: false, error: 'Order not found' });
    // }

    const transaction = await PaystackService.verifyTransaction(reference);

    if (transaction.status !== 'success') {
      return res.json({
        success: false,
        message: `Payment ${transaction.status}`,
        data: transaction,
      });
    }

    // TODO: Verify amount matches
    // if (transaction.amount !== order.amount) {
    //   return res.json({ success: false, message: 'Amount mismatch' });
    // }

    // TODO: Update order
    // await db.orders.update({
    //   where: { reference },
    //   data: { status: 'paid', paidAt: new Date() },
    // });

    return res.json({
      success: true,
      message: 'Payment verified',
      data: {
        reference: transaction.reference,
        amount: transaction.amount,
        status: transaction.status,
        customer: transaction.customer,
      },
    });
  } catch (error) {
    console.error('Verify error:', error);
    return res.status(500).json({
      success: false,
      error: error instanceof Error ? error.message : 'Verification failed',
    });
  }
}

// Handle webhook
export async function handleWebhook(req: Request, res: Response) {
  try {
    // Raw body is available from middleware
    const event = req.body;

    console.log('Webhook event:', event.event);

    // Process based on event type
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
    return res.status(200).json({ received: true });
  } catch (error) {
    console.error('Webhook error:', error);
    // Return 200 to prevent retries
    return res.status(200).json({ received: true });
  }
}

// Event handlers
async function handleChargeSuccess(data: Record<string, unknown>) {
  const reference = data.reference as string;
  const amount = data.amount as number;

  console.log(`Payment successful: ${reference}, Amount: ₦${amount / 100}`);

  // TODO: Update database, send email, fulfill order
}

async function handleChargeFailed(data: Record<string, unknown>) {
  const reference = data.reference as string;
  const reason = data.gateway_response as string;

  console.log(`Payment failed: ${reference}, Reason: ${reason}`);
}

async function handleSubscriptionCreate(data: Record<string, unknown>) {
  const subscriptionCode = data.subscription_code as string;
  const customerEmail = (data.customer as { email: string }).email;

  console.log(`Subscription created: ${subscriptionCode} for ${customerEmail}`);

  // TODO: Activate user subscription
}

async function handleSubscriptionDisable(data: Record<string, unknown>) {
  const subscriptionCode = data.subscription_code as string;

  console.log(`Subscription cancelled: ${subscriptionCode}`);

  // TODO: Revoke access
}

async function handleInvoicePaymentFailed(data: Record<string, unknown>) {
  const customerEmail = (data.customer as { email: string }).email;

  console.log(`Invoice payment failed for ${customerEmail}`);

  // TODO: Notify user
}

// Create subscription plan
export async function createPlan(req: Request, res: Response) {
  try {
    const { name, amount, interval, description } = req.body;

    if (!name || !amount || !interval) {
      return res.status(400).json({
        success: false,
        error: 'Name, amount, and interval are required',
      });
    }

    const plan = await PaystackService.createPlan({
      name,
      amount: PaystackService.toKobo(amount),
      interval,
      description,
    });

    return res.json({
      success: true,
      data: plan,
    });
  } catch (error) {
    console.error('Create plan error:', error);
    return res.status(500).json({
      success: false,
      error: error instanceof Error ? error.message : 'Failed to create plan',
    });
  }
}

// Cancel subscription
export async function cancelSubscription(req: Request, res: Response) {
  try {
    const { code, token } = req.body;

    if (!code || !token) {
      return res.status(400).json({
        success: false,
        error: 'Subscription code and token are required',
      });
    }

    await PaystackService.disableSubscription(code, token);

    return res.json({
      success: true,
      message: 'Subscription cancelled',
    });
  } catch (error) {
    console.error('Cancel subscription error:', error);
    return res.status(500).json({
      success: false,
      error: error instanceof Error ? error.message : 'Failed to cancel',
    });
  }
}
```

---

## Routes

### src/routes/paystack.routes.ts

```typescript
import { Router } from 'express';
import * as PaystackController from '../controllers/paystack.controller';
import { verifyPaystackWebhook } from '../middleware/paystack.middleware';

const router = Router();

// Payment routes
router.post('/initialize', PaystackController.initializePayment);
router.get('/verify', PaystackController.verifyPayment);

// Webhook - uses raw body middleware
router.post('/webhook', verifyPaystackWebhook, PaystackController.handleWebhook);

// Plan routes
router.post('/plan', PaystackController.createPlan);

// Subscription routes
router.post('/subscription/cancel', PaystackController.cancelSubscription);

export default router;
```

---

## Middleware

### src/middleware/paystack.middleware.ts

```typescript
import { Request, Response, NextFunction } from 'express';
import { verifyWebhookSignature } from '../services/paystack.service';

// Extend Request to include rawBody
declare global {
  namespace Express {
    interface Request {
      rawBody?: Buffer;
    }
  }
}

// Middleware to capture raw body (must be used before json parser for webhook route)
export function captureRawBody(req: Request, res: Response, buf: Buffer) {
  req.rawBody = buf;
}

// Middleware to verify Paystack webhook signature
export function verifyPaystackWebhook(
  req: Request,
  res: Response,
  next: NextFunction
) {
  const signature = req.headers['x-paystack-signature'] as string;

  if (!signature) {
    console.error('Missing webhook signature');
    return res.status(401).json({ error: 'Missing signature' });
  }

  const rawBody = req.rawBody;

  if (!rawBody) {
    console.error('Missing raw body');
    return res.status(400).json({ error: 'Missing body' });
  }

  if (!verifyWebhookSignature(rawBody, signature)) {
    console.error('Invalid webhook signature');
    return res.status(401).json({ error: 'Invalid signature' });
  }

  // Parse body if not already parsed
  if (typeof req.body !== 'object' || Object.keys(req.body).length === 0) {
    try {
      req.body = JSON.parse(rawBody.toString('utf8'));
    } catch {
      return res.status(400).json({ error: 'Invalid JSON' });
    }
  }

  next();
}
```

---

## Complete Example

### src/app.ts

```typescript
import express from 'express';
import dotenv from 'dotenv';
import paystackRoutes from './routes/paystack.routes';
import { captureRawBody } from './middleware/paystack.middleware';

// Load environment variables
dotenv.config();

const app = express();

// Capture raw body for webhook verification
// Must be before json() middleware
app.use(
  express.json({
    verify: (req, res, buf) => {
      (req as any).rawBody = buf;
    },
  })
);

// CORS (if needed)
app.use((req, res, next) => {
  res.header('Access-Control-Allow-Origin', process.env.FRONTEND_URL || '*');
  res.header('Access-Control-Allow-Headers', 'Content-Type, Authorization');
  res.header('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE');
  next();
});

// Routes
app.use('/api/paystack', paystackRoutes);

// Health check
app.get('/health', (req, res) => {
  res.json({ status: 'ok' });
});

// Error handler
app.use((err: Error, req: express.Request, res: express.Response, next: express.NextFunction) => {
  console.error('Unhandled error:', err);
  res.status(500).json({ error: 'Internal server error' });
});

// Start server
const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});

export default app;
```

### Usage Example

```bash
# Initialize payment
curl -X POST http://localhost:3000/api/paystack/initialize \
  -H "Content-Type: application/json" \
  -d '{
    "email": "customer@example.com",
    "amount": 5000,
    "metadata": { "order_id": "ORD-123" }
  }'

# Response
{
  "success": true,
  "data": {
    "authorization_url": "https://checkout.paystack.com/xxx",
    "access_code": "xxx",
    "reference": "PAY_XXX"
  }
}

# Verify payment
curl "http://localhost:3000/api/paystack/verify?reference=PAY_XXX"

# Create plan
curl -X POST http://localhost:3000/api/paystack/plan \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Pro Monthly",
    "amount": 9999,
    "interval": "monthly"
  }'
```

### Testing Webhook Locally

```bash
# 1. Install ngrok
npm install -g ngrok

# 2. Start your server
npm run dev

# 3. Expose with ngrok
ngrok http 3000

# 4. Copy the HTTPS URL (e.g., https://abc123.ngrok.io)
# 5. Set in Paystack Dashboard: https://abc123.ngrok.io/api/paystack/webhook

# 6. Make a test payment to trigger webhook
```

### Integration with Prisma

```typescript
// src/services/paystack.service.ts
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

export async function processChargeSuccess(data: {
  reference: string;
  amount: number;
  customer: { email: string };
  authorization?: { authorization_code: string; reusable: boolean };
}) {
  // Update order
  const order = await prisma.order.update({
    where: { reference: data.reference },
    data: {
      status: 'paid',
      paidAt: new Date(),
    },
  });

  // Save card for future use
  if (data.authorization?.reusable) {
    await prisma.savedCard.upsert({
      where: {
        userId_authorizationCode: {
          userId: order.userId,
          authorizationCode: data.authorization.authorization_code,
        },
      },
      update: {},
      create: {
        userId: order.userId,
        authorizationCode: data.authorization.authorization_code,
        email: data.customer.email,
      },
    });
  }

  return order;
}
```
