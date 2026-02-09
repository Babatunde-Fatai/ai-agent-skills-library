> **When to read:** When you need Flutterwave-specific details like encryption key handling, v3 vs v4 API differences, or country-specific payment channels.
> **What problem it solves:** Documents Flutterwave-unique features not found in other payment gateways.
> **When to skip:** If you're implementing basic payments and don't need advanced features.
> **Prerequisites:** Read AGENT_EXECUTION_SPEC.md first.

# Flutterwave-Specific Features

This guide covers features unique to Flutterwave: encryption key handling, API versions, direct card charges, and country-specific considerations.

## Table of Contents

1. [API Versions (v3 vs v4)](#api-versions-v3-vs-v4)
2. [Encryption Key (enckey)](#encryption-key)
3. [Direct Card Charges](#direct-card-charges)
4. [Country-Specific Channels](#country-specific-channels)
5. [Split Payments](#split-payments)
6. [Virtual Accounts](#virtual-accounts)

---

## API Versions (v3 vs v4)

### Version Comparison

| Feature | v3 (Current) | v4 (Beta) |
|---------|--------------|-----------|
| Base URL | `api.flutterwave.com/v3` | `api.flutterwave.com/v4` |
| Authentication | Static API keys | OAuth 2.0 |
| Card Encryption | Full payload encryption | Card-only encryption |
| Error Format | Generic messages | Structured with type/code/message |
| Testing | Limited | X-Scenario-Key header |
| Status | Stable, production-ready | Beta, use with caution |

### v3 API (Recommended)

Use v3 for production applications. It's stable and well-documented.

```typescript
const FLW_BASE_URL = 'https://api.flutterwave.com/v3';

const headers = {
  Authorization: `Bearer ${process.env.FLW_SECRET_KEY}`,
  'Content-Type': 'application/json',
};
```

### v4 API (Beta)

v4 introduces improvements but is still in beta. Key differences:

```typescript
// v4 uses OAuth 2.0
const FLW_V4_BASE_URL = 'https://api.flutterwave.com/v4';

// v4 error responses are structured
interface V4Error {
  type: string;      // Error category
  code: string;      // Specific error code
  message: string;   // Human-readable message
}

// v4 testing with scenario headers
const headers = {
  Authorization: `Bearer ${process.env.FLW_SECRET_KEY}`,
  'Content-Type': 'application/json',
  'X-Scenario-Key': 'payment_failed',  // Simulate specific scenarios
};
```

### Migration Considerations

- v3 will continue to be maintained (no deprecation planned)
- You can only use one version at a time
- Test thoroughly in sandbox before migrating
- v4 reduces boilerplate but requires code changes

---

## Encryption Key

The encryption key (`enckey`) is required for direct card charges where you collect card details on your own form (not recommended for most use cases).

### When You Need enckey

- Direct card charges (you handle card collection)
- PCI DSS compliance required
- Custom card collection UI

### When You DON'T Need enckey

- Using hosted payment page (redirect)
- Using Flutterwave Inline.js
- Mobile money, bank transfers, USSD

### Getting Your Encryption Key

1. Go to **Flutterwave Dashboard** → **Settings** → **API**
2. Find your **Encryption Key** (different from secret key)
3. Store as `FLW_ENCRYPTION_KEY` environment variable

### Using Encryption Key

```typescript
import crypto from 'crypto';

/**
 * Encrypt card payload for direct charge
 * @param payload - The card charge payload
 * @param encryptionKey - Your FLW_ENCRYPTION_KEY
 */
function encryptPayload(payload: object, encryptionKey: string): string {
  const text = JSON.stringify(payload);
  const key = encryptionKey;

  // Flutterwave uses 3DES encryption
  const cipher = crypto.createCipheriv(
    'des-ede3',
    Buffer.from(key.slice(0, 24)),
    Buffer.alloc(0)  // No IV for ECB mode
  );

  let encrypted = cipher.update(text, 'utf8', 'base64');
  encrypted += cipher.final('base64');

  return encrypted;
}

// Usage for direct card charge
const cardPayload = {
  card_number: '5531886652142950',
  cvv: '564',
  expiry_month: '09',
  expiry_year: '32',
  currency: 'NGN',
  amount: 500000,
  email: 'customer@example.com',
  tx_ref: 'FLW_TEST_123',
};

const encryptedData = encryptPayload(cardPayload, process.env.FLW_ENCRYPTION_KEY!);
```

**Warning:** Direct card charges require PCI DSS compliance. Use hosted payment page or Inline.js for most applications.

---

## Direct Card Charges

Direct card charges let you collect card details on your own form. This requires:
- PCI DSS compliance
- Encryption key
- Handling 3DS/PIN authentication flows

### Charge Flow

```
┌───────────────┐     ┌───────────────┐     ┌───────────────┐
│ Collect Card  │────▶│ Encrypt &     │────▶│ Check Auth    │
│ Details       │     │ Charge        │     │ Mode          │
└───────────────┘     └───────────────┘     └───────────────┘
                                                   │
                            ┌──────────────────────┼──────────────────────┐
                            ▼                      ▼                      ▼
                    ┌───────────────┐     ┌───────────────┐     ┌───────────────┐
                    │ PIN Mode:     │     │ 3DS Mode:     │     │ No Auth:      │
                    │ Collect PIN   │     │ Redirect      │     │ Verify        │
                    │ + OTP         │     │ to Bank       │     │ Directly      │
                    └───────────────┘     └───────────────┘     └───────────────┘
```

### Implementation

```typescript
interface DirectChargeRequest {
  card_number: string;
  cvv: string;
  expiry_month: string;
  expiry_year: string;
  currency: string;
  amount: number;
  email: string;
  tx_ref: string;
  fullname?: string;
  phone_number?: string;
}

interface ChargeResponse {
  status: 'success' | 'error';
  message: string;
  data?: {
    id: number;
    tx_ref: string;
    flw_ref: string;
    status: string;
    auth_model: string;
    processor_response: string;
  };
  meta?: {
    authorization: {
      mode: 'pin' | 'redirect' | 'otp' | 'avs_noauth';
      redirect?: string;
      fields?: string[];
    };
  };
}

async function chargeCard(params: DirectChargeRequest): Promise<ChargeResponse> {
  // Encrypt the payload
  const encrypted = encryptPayload(params, process.env.FLW_ENCRYPTION_KEY!);

  const response = await fetch(`${FLW_BASE_URL}/charges?type=card`, {
    method: 'POST',
    headers: getHeaders(),
    body: JSON.stringify({ client: encrypted }),
  });

  return response.json();
}

async function handleChargeResponse(response: ChargeResponse) {
  if (response.status !== 'success') {
    throw new Error(response.message);
  }

  const authMode = response.meta?.authorization?.mode;

  switch (authMode) {
    case 'pin':
      // Collect PIN from customer and submit
      return { mode: 'pin', flwRef: response.data!.flw_ref };

    case 'redirect':
      // Redirect customer to 3DS page
      return { mode: 'redirect', url: response.meta!.authorization!.redirect };

    case 'otp':
      // Collect OTP from customer
      return { mode: 'otp', flwRef: response.data!.flw_ref };

    case 'avs_noauth':
      // Collect AVS details (address verification)
      return { mode: 'avs', fields: response.meta!.authorization!.fields };

    default:
      // No additional auth required
      return { mode: 'complete', transaction: response.data };
  }
}

// Submit PIN
async function submitPin(flwRef: string, pin: string): Promise<ChargeResponse> {
  const response = await fetch(`${FLW_BASE_URL}/validate-charge`, {
    method: 'POST',
    headers: getHeaders(),
    body: JSON.stringify({
      otp: pin,
      flw_ref: flwRef,
      type: 'pin',
    }),
  });

  return response.json();
}

// Submit OTP
async function submitOtp(flwRef: string, otp: string): Promise<ChargeResponse> {
  const response = await fetch(`${FLW_BASE_URL}/validate-charge`, {
    method: 'POST',
    headers: getHeaders(),
    body: JSON.stringify({
      otp: otp,
      flw_ref: flwRef,
      type: 'otp',
    }),
  });

  return response.json();
}
```

---

## Country-Specific Channels

### Nigeria (NG)

| Channel | API Type | Currency |
|---------|----------|----------|
| Card | `card` | NGN |
| Bank Transfer | `bank_transfer` | NGN |
| USSD | `ussd` | NGN |
| Bank Account | `account` | NGN |

### Ghana (GH)

| Channel | API Type | Currency |
|---------|----------|----------|
| Card | `card` | GHS |
| Mobile Money | `mobile_money_ghana` | GHS |
| Networks: MTN, Vodafone, Airtel |

### Kenya (KE)

| Channel | API Type | Currency |
|---------|----------|----------|
| Card | `card` | KES |
| M-Pesa | `mpesa` | KES |

### Uganda (UG)

| Channel | API Type | Currency |
|---------|----------|----------|
| Mobile Money | `mobile_money_uganda` | UGX |
| Networks: MTN, Airtel |

### South Africa (ZA)

| Channel | API Type | Currency |
|---------|----------|----------|
| Card | `card` | ZAR |

### Francophone Africa

| Countries | API Type | Currencies |
|-----------|----------|------------|
| Senegal, Ivory Coast, Mali | `mobile_money_franco` | XOF |
| Cameroon | `mobile_money_franco` | XAF |

### Restricting Payment Options

```typescript
// Only allow specific payment methods
const payment = await initializePayment({
  tx_ref: txRef,
  amount: amount,
  currency: 'NGN',
  redirect_url: redirectUrl,
  customer: { email },
  payment_options: 'card,banktransfer',  // Only card and bank transfer
});

// Available options (comma-separated):
// card, banktransfer, ussd, credit, mobilemoneyghana,
// mobilemoneyuganda, mobilemoneyrwanda, mobilemoneyzambia,
// mobilemoneytanzania, mobilemoneyfranco, mpesa, barter
```

---

## Split Payments

Split payments allow you to automatically distribute funds to multiple recipients.

### Create Subaccount

```typescript
interface CreateSubaccountRequest {
  account_bank: string;
  account_number: string;
  business_name: string;
  business_email: string;
  business_contact: string;
  business_contact_mobile: string;
  business_mobile: string;
  country: string;
  split_type: 'percentage' | 'flat';
  split_value: number;
}

async function createSubaccount(params: CreateSubaccountRequest) {
  const response = await fetch(`${FLW_BASE_URL}/subaccounts`, {
    method: 'POST',
    headers: getHeaders(),
    body: JSON.stringify(params),
  });

  const result = await response.json();
  return result.data;  // { id, subaccount_id, ... }
}

// Usage
const subaccount = await createSubaccount({
  account_bank: '044',
  account_number: '0690000040',
  business_name: 'Vendor Store',
  business_email: 'vendor@example.com',
  business_contact: 'John Vendor',
  business_contact_mobile: '08012345678',
  business_mobile: '08012345678',
  country: 'NG',
  split_type: 'percentage',
  split_value: 20,  // Vendor gets 20%
});
```

### Use Subaccount in Payment

```typescript
// Single subaccount
const payment = await initializePayment({
  tx_ref: txRef,
  amount: amount,
  currency: 'NGN',
  redirect_url: redirectUrl,
  customer: { email },
  subaccounts: [
    {
      id: 'RS_xxx',  // Subaccount ID
      transaction_split_ratio: 2,  // Ratio for splitting
    }
  ],
});

// Multiple subaccounts
const payment = await initializePayment({
  tx_ref: txRef,
  amount: amount,
  currency: 'NGN',
  redirect_url: redirectUrl,
  customer: { email },
  subaccounts: [
    { id: 'RS_vendor1', transaction_split_ratio: 3 },
    { id: 'RS_vendor2', transaction_split_ratio: 2 },
  ],
});
```

---

## Virtual Accounts

Generate virtual bank accounts for customers to pay into.

### Create Virtual Account

```typescript
interface VirtualAccountRequest {
  email: string;
  is_permanent: boolean;
  bvn?: string;
  tx_ref?: string;
  phonenumber?: string;
  firstname?: string;
  lastname?: string;
  narration?: string;
}

async function createVirtualAccount(params: VirtualAccountRequest) {
  const response = await fetch(`${FLW_BASE_URL}/virtual-account-numbers`, {
    method: 'POST',
    headers: getHeaders(),
    body: JSON.stringify(params),
  });

  const result = await response.json();
  return result.data;
}

// Create permanent virtual account
const account = await createVirtualAccount({
  email: 'customer@example.com',
  is_permanent: true,
  bvn: '22222222222',  // Required for permanent accounts
  firstname: 'John',
  lastname: 'Doe',
  narration: 'John Doe Virtual Account',
});

// Response:
// {
//   account_number: '7825000001',
//   bank_name: 'Wema Bank',
//   order_ref: 'URF_xxx',
//   ...
// }
```

### Temporary vs Permanent

| Type | Duration | BVN Required | Use Case |
|------|----------|--------------|----------|
| Temporary | ~30 minutes | No | One-time payments |
| Permanent | Indefinite | Yes | Repeat customers, wallets |

### Get Virtual Account

```typescript
async function getVirtualAccount(orderRef: string) {
  const response = await fetch(
    `${FLW_BASE_URL}/virtual-account-numbers/${orderRef}`,
    { headers: getHeaders() }
  );

  const result = await response.json();
  return result.data;
}
```

---

## Key Differences from Paystack

| Feature | Flutterwave | Paystack |
|---------|-------------|----------|
| Reference | `tx_ref` (you generate) | `reference` (optional, Paystack generates) |
| Webhook header | `verif-hash` | `x-paystack-signature` |
| Hash algorithm | SHA256 (HMAC) or direct | SHA512 (HMAC) |
| Transaction status | `successful`, `pending`, `success-pending-validation` | `success`, `failed`, `abandoned` |
| Verify endpoint | `/transactions/{id}/verify` or by `tx_ref` | `/transaction/verify/{reference}` |
| Initialize endpoint | `/v3/payments` | `/transaction/initialize` |
| Response structure | `{ status: 'success', data: {...} }` | `{ status: true, data: {...} }` |
| Mobile money | Built-in types per country | Via `mobile_money` channel |
| Countries | 34+ African countries | Nigeria, Ghana, Kenya, South Africa |
