> **When to read:** When implementing bank account verification, bank transfers/payouts, or receiving payments via bank transfer.
> **What problem it solves:** Shows account verification, transfer initiation, and bank transfer collection.
> **When to skip:** If you only implement card payments or mobile money.
> **Prerequisites:** Read AGENT_EXECUTION_SPEC.md first.

# Bank Transfers - Complete Guide

This guide covers bank-related operations: account verification, transfers (payouts), and collecting payments via bank transfer.

## Table of Contents

1. [Account Verification](#account-verification)
2. [Bank List](#bank-list)
3. [Transfer to Bank Account (Payouts)](#transfer-to-bank-account)
4. [Collect via Bank Transfer (PWBT)](#collect-via-bank-transfer)
5. [Webhook Handling](#webhook-handling)

---

## Account Verification

Verify bank account details before making transfers to prevent sending money to wrong accounts.

### API Endpoint

```
POST https://api.flutterwave.com/v3/accounts/resolve
```

### Request Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `account_number` | string | Yes | Bank account number |
| `account_bank` | string | Yes | Bank code (see bank list) |

### Implementation

```typescript
interface ResolveAccountRequest {
  account_number: string;
  account_bank: string;
}

interface ResolveAccountResponse {
  status: 'success' | 'error';
  message: string;
  data: {
    account_number: string;
    account_name: string;
  };
}

async function resolveAccount(
  accountNumber: string,
  bankCode: string
): Promise<{ accountNumber: string; accountName: string }> {
  const response = await fetch(`${FLW_BASE_URL}/accounts/resolve`, {
    method: 'POST',
    headers: getHeaders(),
    body: JSON.stringify({
      account_number: accountNumber,
      account_bank: bankCode,
    }),
  });

  const result: ResolveAccountResponse = await response.json();

  if (result.status !== 'success') {
    throw new Error(result.message);
  }

  return {
    accountNumber: result.data.account_number,
    accountName: result.data.account_name,
  };
}

// Usage
const account = await resolveAccount('0690000034', '044');
console.log(account.accountName);  // "DAMILOLA OGUNWALE"
```

### Supported Countries

Account verification is available for:
- Nigeria (NG)
- Ghana (GH)
- Kenya (KE)
- Uganda (UG)
- Tanzania (TZ)
- South Africa (ZA)

---

## Bank List

Get list of supported banks for a country.

### API Endpoint

```
GET https://api.flutterwave.com/v3/banks/{country}
```

### Implementation

```typescript
interface Bank {
  id: number;
  code: string;
  name: string;
}

interface BankListResponse {
  status: 'success' | 'error';
  message: string;
  data: Bank[];
}

async function getBanks(country: string = 'NG'): Promise<Bank[]> {
  const response = await fetch(`${FLW_BASE_URL}/banks/${country}`, {
    headers: getHeaders(),
  });

  const result: BankListResponse = await response.json();

  if (result.status !== 'success') {
    throw new Error(result.message);
  }

  return result.data;
}

// Usage
const nigerianBanks = await getBanks('NG');
// [
//   { id: 1, code: '044', name: 'Access Bank' },
//   { id: 2, code: '023', name: 'Citibank Nigeria' },
//   { id: 3, code: '063', name: 'Diamond Bank' },
//   ...
// ]
```

### Common Nigerian Bank Codes

| Bank Name | Code |
|-----------|------|
| Access Bank | 044 |
| First Bank | 011 |
| GTBank | 058 |
| UBA | 033 |
| Zenith Bank | 057 |
| Kuda Bank | 50211 |
| OPay | 999992 |

---

## Transfer to Bank Account

Send money from your Flutterwave balance to a bank account.

### Prerequisites

1. Enable **Transfer via API** in Dashboard → Settings → API Settings
2. Whitelist server IP addresses for security

### API Endpoint

```
POST https://api.flutterwave.com/v3/transfers
```

### Request Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `account_bank` | string | Yes | Bank code |
| `account_number` | string | Yes | Recipient account number |
| `amount` | number | Yes | Amount in smallest unit |
| `currency` | string | Yes | Currency code (NGN, GHS, etc.) |
| `narration` | string | No | Transfer description |
| `reference` | string | Yes | Your unique reference |
| `debit_currency` | string | No | Currency to debit from your balance |

### Implementation

```typescript
interface TransferRequest {
  account_bank: string;
  account_number: string;
  amount: number;
  currency: string;
  narration?: string;
  reference: string;
  debit_currency?: string;
  meta?: Record<string, unknown>;
}

interface TransferResponse {
  status: 'success' | 'error';
  message: string;
  data: {
    id: number;
    account_number: string;
    bank_code: string;
    full_name: string;
    created_at: string;
    currency: string;
    debit_currency: string;
    amount: number;
    fee: number;
    status: 'NEW' | 'PENDING' | 'SUCCESSFUL' | 'FAILED';
    reference: string;
    meta: Record<string, unknown> | null;
    narration: string;
    complete_message: string;
    requires_approval: number;
    is_approved: number;
    bank_name: string;
  };
}

async function initiateTransfer(
  params: TransferRequest
): Promise<TransferResponse['data']> {
  const response = await fetch(`${FLW_BASE_URL}/transfers`, {
    method: 'POST',
    headers: getHeaders(),
    body: JSON.stringify(params),
  });

  const result: TransferResponse = await response.json();

  if (result.status !== 'success') {
    throw new Error(result.message);
  }

  return result.data;
}

// Usage
function generateTransferRef(): string {
  const timestamp = Date.now().toString(36);
  const random = crypto.randomBytes(4).toString('hex');
  return `TRF_${timestamp}_${random}`.toUpperCase();
}

const transfer = await initiateTransfer({
  account_bank: '044',           // Access Bank
  account_number: '0690000040',
  amount: toSmallestUnit(5000),  // 5,000 NGN
  currency: 'NGN',
  narration: 'Payment for services',
  reference: generateTransferRef(),
});

console.log(transfer.status);  // 'NEW' or 'PENDING'
console.log(transfer.id);      // Use to track status
```

### Transfer Status Values

| Status | Meaning | Action |
|--------|---------|--------|
| `NEW` | Just created | Wait for processing |
| `PENDING` | Being processed | Wait for webhook |
| `SUCCESSFUL` | Transfer completed | Update records |
| `FAILED` | Transfer failed | Retry or refund |

### Verify Transfer Status

```typescript
async function getTransferStatus(transferId: number): Promise<string> {
  const response = await fetch(`${FLW_BASE_URL}/transfers/${transferId}`, {
    headers: getHeaders(),
  });

  const result = await response.json();

  if (result.status !== 'success') {
    throw new Error(result.message);
  }

  return result.data.status;
}
```

---

## Collect via Bank Transfer (PWBT)

Allow customers to pay by transferring to a generated bank account.

### How It Works

1. Customer initiates payment
2. You call Flutterwave to generate a virtual account
3. Customer transfers money to the virtual account
4. Flutterwave notifies you via webhook when transfer is received

### API Endpoint

```
POST https://api.flutterwave.com/v3/charges?type=bank_transfer
```

### Request Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `tx_ref` | string | Yes | Your unique reference |
| `amount` | number | Yes | Expected amount |
| `email` | string | Yes | Customer email |
| `currency` | string | Yes | NGN or GHS |
| `is_permanent` | boolean | No | Create permanent account |

### Implementation

```typescript
interface BankTransferChargeRequest {
  tx_ref: string;
  amount: number;
  email: string;
  currency: string;
  is_permanent?: boolean;
  narration?: string;
}

interface BankTransferChargeResponse {
  status: 'success' | 'error';
  message: string;
  meta: {
    authorization: {
      transfer_reference: string;
      transfer_account: string;
      transfer_bank: string;
      account_expiration: string;
      transfer_note: string;
      transfer_amount: number;
      mode: 'banktransfer';
    };
  };
}

async function initiateBankTransferCharge(
  params: BankTransferChargeRequest
): Promise<BankTransferChargeResponse['meta']['authorization']> {
  const response = await fetch(`${FLW_BASE_URL}/charges?type=bank_transfer`, {
    method: 'POST',
    headers: getHeaders(),
    body: JSON.stringify(params),
  });

  const result: BankTransferChargeResponse = await response.json();

  if (result.status !== 'success') {
    throw new Error(result.message);
  }

  return result.meta.authorization;
}

// Usage
const txRef = generateTxRef('PWBT');

// Store order first
// await db.orders.create({ data: { txRef, amount, status: 'pending' } });

const bankDetails = await initiateBankTransferCharge({
  tx_ref: txRef,
  amount: toSmallestUnit(10000),  // 10,000 NGN
  email: 'customer@example.com',
  currency: 'NGN',
});

// Show these to customer
console.log('Bank:', bankDetails.transfer_bank);
console.log('Account:', bankDetails.transfer_account);
console.log('Amount:', bankDetails.transfer_amount);
console.log('Expires:', bankDetails.account_expiration);
console.log('Reference:', bankDetails.transfer_reference);
```

### Response to Show Customer

```typescript
// Frontend display
{
  "message": "Please transfer to this account",
  "bank": "Wema Bank",
  "accountNumber": "7825000123",
  "amount": 10000,
  "expiresAt": "2024-01-15T12:00:00Z",
  "reference": "PWBT_XXX"
}
```

### Available Currencies

| Currency | Country | Available |
|----------|---------|-----------|
| NGN | Nigeria | Yes |
| GHS | Ghana | Yes |

---

## Webhook Handling

### Transfer Completed

```typescript
interface TransferCompletedEvent {
  event: 'transfer.completed';
  data: {
    id: number;
    account_number: string;
    bank_name: string;
    bank_code: string;
    full_name: string;
    created_at: string;
    currency: string;
    amount: number;
    fee: number;
    status: 'SUCCESSFUL';
    reference: string;
    narration: string;
    complete_message: string;
  };
}

async function handleTransferCompleted(data: TransferCompletedEvent['data']) {
  const reference = data.reference;

  // Update your transfer record
  // await db.transfers.update({
  //   where: { reference },
  //   data: {
  //     status: 'completed',
  //     completedAt: new Date(),
  //   },
  // });

  console.log(`Transfer ${reference} completed: ${data.amount} ${data.currency}`);
}
```

### Transfer Failed

```typescript
interface TransferFailedEvent {
  event: 'transfer.failed';
  data: {
    id: number;
    account_number: string;
    bank_name: string;
    amount: number;
    currency: string;
    status: 'FAILED';
    reference: string;
    complete_message: string;  // Failure reason
  };
}

async function handleTransferFailed(data: TransferFailedEvent['data']) {
  const reference = data.reference;
  const reason = data.complete_message;

  // Update your transfer record
  // await db.transfers.update({
  //   where: { reference },
  //   data: {
  //     status: 'failed',
  //     failureReason: reason,
  //   },
  // });

  // Notify admin or retry
  console.log(`Transfer ${reference} failed: ${reason}`);
}
```

### Bank Transfer Collection (PWBT) Webhook

When customer completes the bank transfer:

```typescript
interface ChargeCompletedEvent {
  event: 'charge.completed';
  data: {
    id: number;
    tx_ref: string;
    flw_ref: string;
    amount: number;
    currency: string;
    status: 'successful';
    payment_type: 'bank_transfer';
    customer: {
      email: string;
    };
  };
}

async function handleBankTransferPayment(data: ChargeCompletedEvent['data']) {
  if (data.payment_type !== 'bank_transfer') return;

  const txRef = data.tx_ref;

  // Find and update order
  // const order = await db.orders.findUnique({ where: { txRef } });
  // if (order && order.status !== 'paid') {
  //   await db.orders.update({
  //     where: { txRef },
  //     data: { status: 'paid', paidAt: new Date() },
  //   });
  // }

  console.log(`Bank transfer payment received: ${txRef}`);
}
```

---

## Complete Payout Flow Example

```typescript
// 1. Verify recipient account
const account = await resolveAccount('0690000040', '044');
console.log(`Sending to: ${account.accountName}`);

// 2. Confirm with user before proceeding
// (Show account name to user for confirmation)

// 3. Initiate transfer
const reference = generateTransferRef();

// Store transfer record first
// await db.transfers.create({
//   data: {
//     reference,
//     recipientAccount: '0690000040',
//     recipientBank: '044',
//     recipientName: account.accountName,
//     amount: toSmallestUnit(50000),
//     currency: 'NGN',
//     status: 'pending',
//   },
// });

const transfer = await initiateTransfer({
  account_bank: '044',
  account_number: '0690000040',
  amount: toSmallestUnit(50000),
  currency: 'NGN',
  narration: 'Withdrawal request #123',
  reference,
});

// 4. Wait for webhook
// transfer.completed or transfer.failed will arrive

// 5. Handle webhook and update records
```

---

## Error Handling

### Common Transfer Errors

| Error | Cause | Solution |
|-------|-------|----------|
| `Invalid account number` | Wrong account number | Verify with resolve endpoint first |
| `Insufficient balance` | Not enough funds | Top up Flutterwave balance |
| `Transfer limit exceeded` | Daily limit reached | Wait or increase limit in dashboard |
| `Invalid bank code` | Wrong bank code | Use bank list endpoint |

### Retry Strategy

```typescript
async function transferWithRetry(
  params: TransferRequest,
  maxRetries = 3
): Promise<TransferResponse['data']> {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await initiateTransfer(params);
    } catch (error) {
      const message = error instanceof Error ? error.message : 'Unknown error';

      // Don't retry certain errors
      if (message.includes('Invalid account') || message.includes('Insufficient')) {
        throw error;
      }

      if (attempt === maxRetries) {
        throw error;
      }

      // Exponential backoff
      await new Promise(r => setTimeout(r, 1000 * Math.pow(2, attempt)));
    }
  }

  throw new Error('Max retries exceeded');
}
```
