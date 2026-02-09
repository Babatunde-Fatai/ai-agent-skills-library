> **When to read:** When implementing mobile money payments (M-Pesa, MTN, Airtel, Vodafone).
> **What problem it solves:** Shows charge flow for mobile money across different African countries.
> **When to skip:** If you only implement card payments or bank transfers.
> **Prerequisites:** Read AGENT_EXECUTION_SPEC.md first.

# Mobile Money - Complete Guide

This guide covers mobile money payment collection across African countries including Kenya (M-Pesa), Ghana (MTN, Vodafone, Airtel), Uganda (MTN, Airtel), and Francophone Africa.

## Table of Contents

1. [Overview](#overview)
2. [Supported Countries & Networks](#supported-countries--networks)
3. [Kenya (M-Pesa)](#kenya-m-pesa)
4. [Ghana Mobile Money](#ghana-mobile-money)
5. [Uganda Mobile Money](#uganda-mobile-money)
6. [Francophone Mobile Money](#francophone-mobile-money)
7. [Webhook Handling](#webhook-handling)
8. [Common Patterns](#common-patterns)

---

## Overview

Mobile money allows customers to pay using their mobile money wallets (M-Pesa, MTN, Airtel, etc.). The flow typically involves:

1. Customer provides their phone number
2. You initiate a charge with Flutterwave
3. Customer receives a prompt on their phone
4. Customer enters PIN to authorize
5. Flutterwave confirms payment via webhook

### General Flow

```
┌─────────┐     ┌─────────┐     ┌───────────┐     ┌─────────┐
│ Customer│────▶│ Your App│────▶│Flutterwave│────▶│ Mobile  │
│  (phone)│     │ (charge)│     │   (API)   │     │ Money   │
└─────────┘     └─────────┘     └───────────┘     └─────────┘
                                      │                │
                                      │ Push prompt    │
                                      │◀───────────────│
                                      │                │
                                      │ PIN authorized │
                                      │◀───────────────│
                                      │                │
                              Webhook │                │
                              ───────▶│                │
```

---

## Supported Countries & Networks

| Country | Currency | Networks | Endpoint Type |
|---------|----------|----------|---------------|
| Kenya | KES | M-Pesa, Airtel | `mpesa` |
| Ghana | GHS | MTN, Vodafone, Airtel | `mobile_money_ghana` |
| Uganda | UGX | MTN, Airtel | `mobile_money_uganda` |
| Rwanda | RWF | MTN, Airtel | `mobile_money_rwanda` |
| Zambia | ZMW | MTN, Airtel | `mobile_money_zambia` |
| Tanzania | TZS | M-Pesa, Airtel, Tigo | `mobile_money_tanzania` |
| Francophone | XOF/XAF | Orange, MTN, Moov | `mobile_money_franco` |

---

## Kenya (M-Pesa)

### API Endpoint

```
POST https://api.flutterwave.com/v3/charges?type=mpesa
```

### Request Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `tx_ref` | string | Yes | Your unique reference |
| `amount` | number | Yes | Amount in KES (smallest unit) |
| `currency` | string | Yes | "KES" |
| `email` | string | Yes | Customer email |
| `phone_number` | string | Yes | Customer phone (254...) |
| `fullname` | string | No | Customer name |

### Implementation

```typescript
interface MpesaChargeRequest {
  tx_ref: string;
  amount: number;
  currency: 'KES';
  email: string;
  phone_number: string;  // Format: 254XXXXXXXXX
  fullname?: string;
}

interface MpesaChargeResponse {
  status: 'success' | 'error';
  message: string;
  data: {
    id: number;
    tx_ref: string;
    flw_ref: string;
    device_fingerprint: string;
    amount: number;
    charged_amount: number;
    app_fee: number;
    merchant_fee: number;
    processor_response: string;
    auth_model: string;
    currency: string;
    ip: string;
    narration: string;
    status: 'pending' | 'successful' | 'failed';
    payment_type: 'mpesa';
    fraud_status: string;
    created_at: string;
    account_id: number;
    customer: {
      id: number;
      phone_number: string;
      name: string;
      email: string;
      created_at: string;
    };
  };
  meta: {
    authorization: {
      mode: 'callback';
    };
  };
}

async function chargeMpesa(
  params: MpesaChargeRequest
): Promise<MpesaChargeResponse['data']> {
  const response = await fetch(`${FLW_BASE_URL}/charges?type=mpesa`, {
    method: 'POST',
    headers: getHeaders(),
    body: JSON.stringify(params),
  });

  const result: MpesaChargeResponse = await response.json();

  if (result.status !== 'success') {
    throw new Error(result.message);
  }

  return result.data;
}

// Usage
const txRef = generateTxRef('MPESA');

// Store order first
// await db.orders.create({ data: { txRef, amount: 50000, currency: 'KES', status: 'pending' } });

const charge = await chargeMpesa({
  tx_ref: txRef,
  amount: toSmallestUnit(500),  // 500 KES
  currency: 'KES',
  email: 'customer@example.com',
  phone_number: '254712345678',  // Kenyan format
  fullname: 'John Doe',
});

// Customer will receive STK push on their phone
// Wait for webhook or poll for status
console.log(charge.status);  // 'pending'
```

### Phone Number Format

Kenyan numbers must be in international format without the plus sign:
- `254712345678` (correct)
- `+254712345678` (may work)
- `0712345678` (incorrect)

---

## Ghana Mobile Money

### API Endpoint

```
POST https://api.flutterwave.com/v3/charges?type=mobile_money_ghana
```

### Request Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `tx_ref` | string | Yes | Your unique reference |
| `amount` | number | Yes | Amount in GHS |
| `currency` | string | Yes | "GHS" |
| `email` | string | Yes | Customer email |
| `phone_number` | string | Yes | Customer phone (233...) |
| `network` | string | Yes | "MTN", "VODAFONE", or "AIRTEL" |

### Implementation

```typescript
interface GhanaMoMoRequest {
  tx_ref: string;
  amount: number;
  currency: 'GHS';
  email: string;
  phone_number: string;  // Format: 233XXXXXXXXX
  network: 'MTN' | 'VODAFONE' | 'AIRTEL';
}

interface GhanaMoMoResponse {
  status: 'success' | 'error';
  message: string;
  meta: {
    authorization: {
      redirect: string;  // URL to complete payment
      mode: 'redirect';
    };
  };
}

async function chargeGhanaMoMo(
  params: GhanaMoMoRequest
): Promise<string> {
  const response = await fetch(
    `${FLW_BASE_URL}/charges?type=mobile_money_ghana`,
    {
      method: 'POST',
      headers: getHeaders(),
      body: JSON.stringify(params),
    }
  );

  const result: GhanaMoMoResponse = await response.json();

  if (result.status !== 'success') {
    throw new Error(result.message);
  }

  return result.meta.authorization.redirect;
}

// Usage
const txRef = generateTxRef('GH');

const redirectUrl = await chargeGhanaMoMo({
  tx_ref: txRef,
  amount: toSmallestUnit(100),  // 100 GHS
  currency: 'GHS',
  email: 'customer@example.com',
  phone_number: '233551234567',
  network: 'MTN',
});

// Redirect customer to complete payment
// return Response.redirect(redirectUrl);
```

### Network Detection

```typescript
function detectGhanaNetwork(phoneNumber: string): 'MTN' | 'VODAFONE' | 'AIRTEL' | null {
  const number = phoneNumber.replace(/^\+?233/, '');

  // MTN prefixes
  if (/^(24|54|55|59)/.test(number)) return 'MTN';

  // Vodafone prefixes
  if (/^(20|50)/.test(number)) return 'VODAFONE';

  // Airtel prefixes
  if (/^(26|56|57)/.test(number)) return 'AIRTEL';

  return null;
}
```

---

## Uganda Mobile Money

### API Endpoint

```
POST https://api.flutterwave.com/v3/charges?type=mobile_money_uganda
```

### Request Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `tx_ref` | string | Yes | Your unique reference |
| `amount` | number | Yes | Amount in UGX |
| `currency` | string | Yes | "UGX" |
| `email` | string | Yes | Customer email |
| `phone_number` | string | Yes | Customer phone (256...) |
| `network` | string | Yes | "MTN" or "AIRTEL" |

### Implementation

```typescript
interface UgandaMoMoRequest {
  tx_ref: string;
  amount: number;
  currency: 'UGX';
  email: string;
  phone_number: string;  // Format: 256XXXXXXXXX
  network: 'MTN' | 'AIRTEL';
}

async function chargeUgandaMoMo(
  params: UgandaMoMoRequest
): Promise<string> {
  const response = await fetch(
    `${FLW_BASE_URL}/charges?type=mobile_money_uganda`,
    {
      method: 'POST',
      headers: getHeaders(),
      body: JSON.stringify(params),
    }
  );

  const result = await response.json();

  if (result.status !== 'success') {
    throw new Error(result.message);
  }

  return result.meta.authorization.redirect;
}

// Usage
const txRef = generateTxRef('UG');

const redirectUrl = await chargeUgandaMoMo({
  tx_ref: txRef,
  amount: toSmallestUnit(50000),  // 50,000 UGX
  currency: 'UGX',
  email: 'customer@example.com',
  phone_number: '256772123456',
  network: 'MTN',
});
```

---

## Francophone Mobile Money

For Senegal, Ivory Coast, Mali, Cameroon, and other Francophone African countries.

### API Endpoint

```
POST https://api.flutterwave.com/v3/charges?type=mobile_money_franco
```

### Supported Countries & Currencies

| Country | Currency | Networks |
|---------|----------|----------|
| Senegal | XOF | Orange, Free |
| Ivory Coast | XOF | MTN, Orange, Moov |
| Mali | XOF | Orange, Moov |
| Cameroon | XAF | MTN, Orange |
| Burkina Faso | XOF | Orange, Moov |

### Request Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `tx_ref` | string | Yes | Your unique reference |
| `amount` | number | Yes | Amount in XOF/XAF |
| `currency` | string | Yes | "XOF" or "XAF" |
| `email` | string | Yes | Customer email |
| `phone_number` | string | Yes | Customer phone |
| `country` | string | Yes | Country code (SN, CI, ML, CM) |

### Implementation

```typescript
interface FrancoMoMoRequest {
  tx_ref: string;
  amount: number;
  currency: 'XOF' | 'XAF';
  email: string;
  phone_number: string;
  country: 'SN' | 'CI' | 'ML' | 'CM' | 'BF';
}

async function chargeFrancoMoMo(
  params: FrancoMoMoRequest
): Promise<string> {
  const response = await fetch(
    `${FLW_BASE_URL}/charges?type=mobile_money_franco`,
    {
      method: 'POST',
      headers: getHeaders(),
      body: JSON.stringify(params),
    }
  );

  const result = await response.json();

  if (result.status !== 'success') {
    throw new Error(result.message);
  }

  return result.meta.authorization.redirect;
}

// Usage - Senegal
const txRef = generateTxRef('SN');

const redirectUrl = await chargeFrancoMoMo({
  tx_ref: txRef,
  amount: toSmallestUnit(5000),  // 5,000 XOF
  currency: 'XOF',
  email: 'customer@example.com',
  phone_number: '221771234567',  // Senegal
  country: 'SN',
});
```

---

## Webhook Handling

Mobile money payments are asynchronous. Always rely on webhooks for confirmation.

### Webhook Event

```typescript
interface MobileMoneyWebhookEvent {
  event: 'charge.completed';
  data: {
    id: number;
    tx_ref: string;
    flw_ref: string;
    amount: number;
    currency: string;
    charged_amount: number;
    status: 'successful' | 'failed';
    payment_type: 'mpesa' | 'mobilemoneyghana' | 'mobilemoneyuganda' | 'mobilemoneyrwanda';
    customer: {
      id: number;
      email: string;
      phone_number: string;
      name: string;
    };
    created_at: string;
  };
}

async function handleMobileMoneyWebhook(event: MobileMoneyWebhookEvent) {
  const { data } = event;
  const txRef = data.tx_ref;

  // Find order
  const order = await db.orders.findUnique({ where: { txRef } });

  if (!order) {
    console.error(`Order not found: ${txRef}`);
    return;
  }

  // Idempotency check
  if (order.status === 'paid') {
    console.log(`Order ${txRef} already paid`);
    return;
  }

  // Verify status
  if (data.status !== 'successful') {
    console.log(`Payment ${txRef} not successful: ${data.status}`);
    // await db.orders.update({ where: { txRef }, data: { status: 'failed' } });
    return;
  }

  // Verify amount
  if (data.amount !== order.amount) {
    console.error(`Amount mismatch for ${txRef}`);
    return;
  }

  // Mark as paid
  // await db.orders.update({
  //   where: { txRef },
  //   data: { status: 'paid', paidAt: new Date() },
  // });

  console.log(`Mobile money payment confirmed: ${txRef}`);
}
```

---

## Common Patterns

### Unified Mobile Money Charge Function

```typescript
type MobileMoneyCountry = 'KE' | 'GH' | 'UG' | 'RW' | 'ZM' | 'TZ' | 'SN' | 'CI' | 'CM';

interface MobileMoneyChargeParams {
  txRef: string;
  amount: number;
  email: string;
  phoneNumber: string;
  country: MobileMoneyCountry;
  network?: string;
}

const COUNTRY_CONFIG: Record<MobileMoneyCountry, {
  currency: string;
  type: string;
  requiresNetwork: boolean;
}> = {
  KE: { currency: 'KES', type: 'mpesa', requiresNetwork: false },
  GH: { currency: 'GHS', type: 'mobile_money_ghana', requiresNetwork: true },
  UG: { currency: 'UGX', type: 'mobile_money_uganda', requiresNetwork: true },
  RW: { currency: 'RWF', type: 'mobile_money_rwanda', requiresNetwork: true },
  ZM: { currency: 'ZMW', type: 'mobile_money_zambia', requiresNetwork: true },
  TZ: { currency: 'TZS', type: 'mobile_money_tanzania', requiresNetwork: true },
  SN: { currency: 'XOF', type: 'mobile_money_franco', requiresNetwork: false },
  CI: { currency: 'XOF', type: 'mobile_money_franco', requiresNetwork: false },
  CM: { currency: 'XAF', type: 'mobile_money_franco', requiresNetwork: false },
};

async function chargeMobileMoney(params: MobileMoneyChargeParams): Promise<{
  status: 'pending' | 'redirect';
  redirectUrl?: string;
  transactionId?: number;
}> {
  const config = COUNTRY_CONFIG[params.country];

  if (!config) {
    throw new Error(`Unsupported country: ${params.country}`);
  }

  const payload: Record<string, unknown> = {
    tx_ref: params.txRef,
    amount: params.amount,
    currency: config.currency,
    email: params.email,
    phone_number: params.phoneNumber,
  };

  if (config.requiresNetwork && params.network) {
    payload.network = params.network;
  }

  if (['SN', 'CI', 'CM'].includes(params.country)) {
    payload.country = params.country;
  }

  const response = await fetch(
    `${FLW_BASE_URL}/charges?type=${config.type}`,
    {
      method: 'POST',
      headers: getHeaders(),
      body: JSON.stringify(payload),
    }
  );

  const result = await response.json();

  if (result.status !== 'success') {
    throw new Error(result.message);
  }

  // M-Pesa returns data directly, others return redirect
  if (result.meta?.authorization?.mode === 'redirect') {
    return {
      status: 'redirect',
      redirectUrl: result.meta.authorization.redirect,
    };
  }

  return {
    status: 'pending',
    transactionId: result.data.id,
  };
}
```

### Test Mobile Money Numbers

| Country | Phone Number | OTP |
|---------|--------------|-----|
| Ghana | 0551234987 | 123456 |
| Kenya | 254712345678 | N/A (auto-approve in test) |
| Uganda | 256772123456 | 123456 |

---

## Error Handling

### Common Errors

| Error | Cause | Solution |
|-------|-------|----------|
| `Invalid phone number` | Wrong format | Use international format without + |
| `Network not supported` | Wrong network value | Check supported networks |
| `Transaction failed` | Customer didn't authorize | Prompt customer to retry |
| `Insufficient balance` | Not enough in wallet | Customer needs to top up |

### Timeout Handling

Mobile money transactions can take time. Implement polling or rely on webhooks:

```typescript
async function pollForStatus(
  transactionId: number,
  maxAttempts = 30,
  interval = 10000
): Promise<string> {
  for (let attempt = 0; attempt < maxAttempts; attempt++) {
    const response = await fetch(
      `${FLW_BASE_URL}/transactions/${transactionId}/verify`,
      { headers: getHeaders() }
    );

    const result = await response.json();
    const status = result.data?.status;

    if (status === 'successful' || status === 'failed') {
      return status;
    }

    await new Promise(r => setTimeout(r, interval));
  }

  throw new Error('Transaction timed out');
}
```
