> **When to read:** When implementing backend verification and webhook handling in Express.js.
> **What problem it solves:** Maps Agent Execution Spec roles to Express project structure.
> **When to skip:** If you are not using Express/Node or only work on frontend UI.
> **Prerequisites:** Read AGENT_EXECUTION_SPEC.md first.

# Express.js Implementation - Complete Guide

Complete Flutterwave integration for Express.js with TypeScript.

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
│   │   └── flutterwave.ts         # Configuration
│   ├── controllers/
│   │   └── flutterwave.controller.ts  # Route handlers
│   ├── middleware/
│   │   └── flutterwave.middleware.ts  # Webhook verification
│   ├── routes/
│   │   └── flutterwave.routes.ts  # Route definitions
│   ├── services/
│   │   └── flutterwave.service.ts # API calls
│   ├── types/
│   │   └── flutterwave.types.ts   # TypeScript types
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
FLW_SECRET_KEY=FLWSECK_TEST-xxxxxxxxxxxxxxxxxxxx
FLW_PUBLIC_KEY=FLWPUBK_TEST-xxxxxxxxxxxxxxxxxxxx
FLW_SECRET_HASH=your_webhook_secret_hash
FRONTEND_URL=http://localhost:3001
```

### src/config/flutterwave.ts

```typescript
export const FLW_CONFIG = {
  baseUrl: 'https://api.flutterwave.com/v3',
  secretKey: process.env.FLW_SECRET_KEY!,
  publicKey: process.env.FLW_PUBLIC_KEY!,
  secretHash: process.env.FLW_SECRET_HASH!,
};

export const getFlutterwaveHeaders = () => ({
  Authorization: `Bearer ${FLW_CONFIG.secretKey}`,
  'Content-Type': 'application/json',
});
```

---

## Services

### src/services/flutterwave.service.ts

```typescript
import crypto from 'crypto';
import { FLW_CONFIG, getFlutterwaveHeaders } from '../config/flutterwave';

const BASE_URL = FLW_CONFIG.baseUrl;

// Utility functions
export function toSmallestUnit(amount: number): number {
  return Math.round(amount * 100);
}

export function fromSmallestUnit(amount: number): number {
  return amount / 100;
}

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
  const response = await fetch(`${BASE_URL}/payments`, {
    method: 'POST',
    headers: getFlutterwaveHeaders(),
    body: JSON.stringify(params),
  });

  const result = await response.json();

  if (result.status !== 'success') {
    throw new Error(result.message || 'Failed to initialize payment');
  }

  return result.data as { link: string };
}

// Verify transaction by ID
export async function verifyTransaction(transactionId: number) {
  const response = await fetch(
    `${BASE_URL}/transactions/${transactionId}/verify`,
    { headers: getFlutterwaveHeaders() }
  );

  const result = await response.json();

  if (result.status !== 'success') {
    throw new Error(result.message || 'Failed to verify transaction');
  }

  return result.data;
}

// Verify transaction by tx_ref
export async function verifyByTxRef(txRef: string) {
  const response = await fetch(
    `${BASE_URL}/transactions/verify_by_reference?tx_ref=${encodeURIComponent(txRef)}`,
    { headers: getFlutterwaveHeaders() }
  );

  const result = await response.json();

  if (result.status !== 'success') {
    throw new Error(result.message || 'Failed to verify transaction');
  }

  return result.data;
}

// Initiate transfer (payout)
export async function initiateTransfer(params: {
  account_bank: string;
  account_number: string;
  amount: number;
  currency: string;
  narration?: string;
  reference: string;
}) {
  const response = await fetch(`${BASE_URL}/transfers`, {
    method: 'POST',
    headers: getFlutterwaveHeaders(),
    body: JSON.stringify(params),
  });

  const result = await response.json();

  if (result.status !== 'success') {
    throw new Error(result.message || 'Failed to initiate transfer');
  }

  return result.data;
}

// Resolve bank account
export async function resolveAccount(accountNumber: string, bankCode: string) {
  const response = await fetch(`${BASE_URL}/accounts/resolve`, {
    method: 'POST',
    headers: getFlutterwaveHeaders(),
    body: JSON.stringify({
      account_number: accountNumber,
      account_bank: bankCode,
    }),
  });

  const result = await response.json();

  if (result.status !== 'success') {
    throw new Error(result.message || 'Failed to resolve account');
  }

  return result.data;
}

// Verify webhook signature (simple verif-hash method)
export function verifyWebhookSignature(signature: string | undefined): boolean {
  if (!signature || !FLW_CONFIG.secretHash) {
    return false;
  }
  return signature === FLW_CONFIG.secretHash;
}

// Verify webhook signature (HMAC-SHA256 method)
export function verifyWebhookHMAC(
  payload: string | Buffer,
  signature: string
): boolean {
  const payloadString = typeof payload === 'string'
    ? payload
    : payload.toString('utf8');

  const hash = crypto
    .createHmac('sha256', FLW_CONFIG.secretHash)
    .update(payloadString)
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

## Controllers

### src/controllers/flutterwave.controller.ts

```typescript
import { Request, Response } from 'express';
import * as FlutterwaveService from '../services/flutterwave.service';

// Initialize payment
export async function initializePayment(req: Request, res: Response) {
  try {
    const { email, amount, currency = 'NGN', name, metadata } = req.body;

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
        error: 'Minimum amount is 1',
      });
    }

    // STEP 1: Generate unique tx_ref
    const txRef = FlutterwaveService.generateTxRef('PAY');
    const amountInSmallestUnit = FlutterwaveService.toSmallestUnit(amount);

    // STEP 2: Store order in database FIRST
    await db.orders.create({
      data: {
        txRef,
        email,
        amount: amountInSmallestUnit,
        currency,
        status: 'pending',
      },
    });

    // STEP 3: Initialize payment with Flutterwave
    const payment = await FlutterwaveService.initializePayment({
      tx_ref: txRef,
      amount: amountInSmallestUnit,
      currency,
      redirect_url: `${process.env.FRONTEND_URL}/payment/callback`,
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

    return res.json({
      success: true,
      data: {
        link: payment.link,
        txRef,
      },
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
    const { transaction_id, tx_ref } = req.query;

    if (!transaction_id && !tx_ref) {
      return res.status(400).json({
        success: false,
        error: 'transaction_id or tx_ref is required',
      });
    }

    // Verify with Flutterwave
    const transaction = transaction_id
      ? await FlutterwaveService.verifyTransaction(parseInt(transaction_id as string))
      : await FlutterwaveService.verifyByTxRef(tx_ref as string);

    // Get expected amount from database
    const order = await db.orders.findUnique({
      where: { txRef: transaction.tx_ref },
    });
    
    if (!order) {
      return res.status(404).json({ success: false, error: 'Order not found' });
    }

    // Verify status
    if (transaction.status !== 'successful') {
      return res.json({
        success: false,
        message: `Payment ${transaction.status}`,
        data: transaction,
      });
    }

    // CRITICAL: Verify amount matches
    if (transaction.amount !== order.amount) {
      return res.json({ success: false, message: 'Amount mismatch' });
    }

    Update order
    await db.orders.update({
      where: { txRef: transaction.tx_ref },
      data: { status: 'paid', paidAt: new Date() },
    });

    return res.json({
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
    return res.status(500).json({
      success: false,
      error: error instanceof Error ? error.message : 'Verification failed',
    });
  }
}

// Handle webhook
export async function handleWebhook(req: Request, res: Response) {
  try {
    const event = req.body;

    console.log('Webhook event:', event.event);

    // Process based on event type
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
    return res.status(200).json({ received: true });
  } catch (error) {
    console.error('Webhook error:', error);
    // Return 200 to prevent retries
    return res.status(200).json({ received: true });
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

  // Get order and verify
  const order = await db.orders.findUnique({ where: { txRef } });
  if (!order || order.status === 'paid') return;
  if (order.amount !== amount) {
    console.error(`Amount mismatch for ${txRef}`);
    return;
  }

  Update order
  await db.orders.update({
    where: { txRef },
    data: { status: 'paid', paidAt: new Date() },
  });
}

async function handleChargeFailed(data: Record<string, unknown>) {
  const txRef = data.tx_ref as string;
  const reason = data.processor_response as string;
  console.log(`Payment failed: ${txRef} - ${reason}`);
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

// Resolve bank account
export async function resolveBankAccount(req: Request, res: Response) {
  try {
    const { account_number, bank_code } = req.body;

    if (!account_number || !bank_code) {
      return res.status(400).json({
        success: false,
        error: 'account_number and bank_code are required',
      });
    }

    const account = await FlutterwaveService.resolveAccount(
      account_number,
      bank_code
    );

    return res.json({
      success: true,
      data: account,
    });
  } catch (error) {
    console.error('Resolve account error:', error);
    return res.status(500).json({
      success: false,
      error: error instanceof Error ? error.message : 'Failed to resolve account',
    });
  }
}

// Initiate transfer
export async function createTransfer(req: Request, res: Response) {
  try {
    const { account_number, bank_code, amount, currency = 'NGN', narration } = req.body;

    if (!account_number || !bank_code || !amount) {
      return res.status(400).json({
        success: false,
        error: 'account_number, bank_code, and amount are required',
      });
    }

    const reference = FlutterwaveService.generateTxRef('TRF');

    // Store transfer record first
    await db.transfers.create({
      data: { reference, accountNumber: account_number, bankCode: bank_code, amount, status: 'pending' },
    });

    const transfer = await FlutterwaveService.initiateTransfer({
      account_bank: bank_code,
      account_number,
      amount: FlutterwaveService.toSmallestUnit(amount),
      currency,
      narration,
      reference,
    });

    return res.json({
      success: true,
      data: transfer,
    });
  } catch (error) {
    console.error('Transfer error:', error);
    return res.status(500).json({
      success: false,
      error: error instanceof Error ? error.message : 'Failed to initiate transfer',
    });
  }
}
```

---

## Routes

### src/routes/flutterwave.routes.ts

```typescript
import { Router } from 'express';
import * as FlutterwaveController from '../controllers/flutterwave.controller';
import { verifyFlutterwaveWebhook } from '../middleware/flutterwave.middleware';

const router = Router();

// Payment routes
router.post('/initialize', FlutterwaveController.initializePayment);
router.get('/verify', FlutterwaveController.verifyPayment);

// Webhook - uses signature verification middleware
router.post('/webhook', verifyFlutterwaveWebhook, FlutterwaveController.handleWebhook);

// Bank routes
router.post('/resolve-account', FlutterwaveController.resolveBankAccount);
router.post('/transfer', FlutterwaveController.createTransfer);

export default router;
```

---

## Middleware

### src/middleware/flutterwave.middleware.ts

```typescript
import { Request, Response, NextFunction } from 'express';
import { verifyWebhookSignature } from '../services/flutterwave.service';

// Extend Request to include rawBody
declare global {
  namespace Express {
    interface Request {
      rawBody?: Buffer;
    }
  }
}

// Middleware to capture raw body
export function captureRawBody(req: Request, res: Response, buf: Buffer) {
  req.rawBody = buf;
}

// Middleware to verify Flutterwave webhook signature
export function verifyFlutterwaveWebhook(
  req: Request,
  res: Response,
  next: NextFunction
) {
  const signature = req.headers['verif-hash'] as string;

  if (!verifyWebhookSignature(signature)) {
    console.error('Invalid webhook signature');
    return res.status(401).json({ error: 'Invalid signature' });
  }

  // Parse body if not already parsed
  if (req.rawBody && (!req.body || Object.keys(req.body).length === 0)) {
    try {
      req.body = JSON.parse(req.rawBody.toString('utf8'));
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
import flutterwaveRoutes from './routes/flutterwave.routes';
import { captureRawBody } from './middleware/flutterwave.middleware';

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
app.use('/api/flutterwave', flutterwaveRoutes);

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

### Usage Examples

```bash
# Initialize payment
curl -X POST http://localhost:3000/api/flutterwave/initialize \
  -H "Content-Type: application/json" \
  -d '{
    "email": "customer@example.com",
    "amount": 5000,
    "currency": "NGN",
    "name": "John Doe"
  }'

# Response
{
  "success": true,
  "data": {
    "link": "https://checkout.flutterwave.com/v3/hosted/pay/xxx",
    "txRef": "PAY_LK5J2M8_A1B2C3D4"
  }
}

# Verify payment
curl "http://localhost:3000/api/flutterwave/verify?transaction_id=12345"

# Or by tx_ref
curl "http://localhost:3000/api/flutterwave/verify?tx_ref=PAY_LK5J2M8_A1B2C3D4"

# Resolve bank account
curl -X POST http://localhost:3000/api/flutterwave/resolve-account \
  -H "Content-Type: application/json" \
  -d '{
    "account_number": "0690000040",
    "bank_code": "044"
  }'

# Initiate transfer
curl -X POST http://localhost:3000/api/flutterwave/transfer \
  -H "Content-Type: application/json" \
  -d '{
    "account_number": "0690000040",
    "bank_code": "044",
    "amount": 5000,
    "currency": "NGN",
    "narration": "Payment for services"
  }'
```

### Testing Webhook Locally

```bash
# 1. Start server
npm run dev

# 2. Expose with ngrok
ngrok http 3000

# 3. Set webhook URL in Flutterwave Dashboard
# https://abc123.ngrok.io/api/flutterwave/webhook

# 4. Test manually
SECRET_HASH="your_secret_hash"
PAYLOAD='{"event":"charge.completed","data":{"tx_ref":"TEST_123","amount":500000,"status":"successful"}}'

curl -X POST http://localhost:3000/api/flutterwave/webhook \
  -H "Content-Type: application/json" \
  -H "verif-hash: $SECRET_HASH" \
  -d "$PAYLOAD"
```
