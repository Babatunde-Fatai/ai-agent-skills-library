> **When to read:** When implementing subscription plans, recurring billing, or invoice lifecycle.
> **What problem it solves:** Guides plan creation, customer onboarding, and webhook-driven lifecycle management.
> **When to skip:** If you only need one-time payments or only frontend changes.
> **Prerequisites:** Read AGENT_EXECUTION_SPEC.md first.

# Subscriptions & Recurring Billing - Complete Guide

This guide covers subscription plans, customer management, and recurring billing with Paystack.

## Table of Contents

1. [Subscription Flow Overview](#subscription-flow-overview)
2. [Plans](#plans)
3. [Customers](#customers)
4. [Subscriptions](#subscriptions)
5. [Managing Subscriptions](#managing-subscriptions)
6. [Invoices](#invoices)

---

## Subscription Flow Overview

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        SUBSCRIPTION FLOW                                 │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  SETUP (One-time)                                                        │
│  ┌─────────────────┐                                                     │
│  │  Create Plan    │ → plan_code (e.g., PLN_xxx)                        │
│  │  (Dashboard/API)│   Define: name, amount, interval                   │
│  └─────────────────┘                                                     │
│                                                                          │
│  SUBSCRIBE (Per customer)                                                │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐      │
│  │ Initialize with │───▶│ Customer pays   │───▶│ Subscription    │      │
│  │ plan parameter  │    │ first time      │    │ auto-created    │      │
│  └─────────────────┘    └─────────────────┘    └─────────────────┘      │
│                                                                          │
│  LIFECYCLE (Automatic)                                                   │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐      │
│  │ invoice.create  │───▶│ Auto-charge     │───▶│ charge.success  │      │
│  │ (3 days before) │    │ (on due date)   │    │ OR failed       │      │
│  └─────────────────┘    └─────────────────┘    └─────────────────┘      │
│                                                                          │
│  MANAGEMENT                                                              │
│  - Customer: Use email_token link to manage card/cancel                  │
│  - You: Use API to enable/disable/fetch subscriptions                    │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## Plans

Plans define the billing amount and interval. Create them in the Dashboard or via API.

### Create Plan via Dashboard

1. Go to **Paystack Dashboard** → **Products** → **Plans**
2. Click **Create Plan**
3. Fill in: Name, Amount, Interval, Description
4. Save and note the `plan_code` (e.g., `PLN_xxx`)

### Create Plan via API

```
POST https://api.paystack.co/plan
```

```typescript
interface CreatePlanRequest {
  name: string;
  amount: number; // in kobo
  interval: 'daily' | 'weekly' | 'monthly' | 'quarterly' | 'biannually' | 'annually';
  description?: string;
  currency?: 'NGN' | 'GHS' | 'ZAR' | 'KES' | 'USD';
  invoice_limit?: number; // Number of invoices before stopping (0 = unlimited)
  send_invoices?: boolean;
  send_sms?: boolean;
}

interface Plan {
  id: number;
  name: string;
  plan_code: string;
  amount: number;
  interval: string;
  currency: string;
  description: string | null;
  send_invoices: boolean;
  send_sms: boolean;
  hosted_page: boolean;
  hosted_page_url: string | null;
  hosted_page_summary: string | null;
  is_archived: boolean;
  created_at: string;
  updated_at: string;
}

async function createPlan(params: CreatePlanRequest): Promise<Plan> {
  const response = await fetch('https://api.paystack.co/plan', {
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
const plan = await createPlan({
  name: 'Pro Monthly',
  amount: 500000, // 5,000 NGN
  interval: 'monthly',
  description: 'Pro plan with all features',
});

console.log(plan.plan_code); // PLN_xxx - use this for subscriptions
```

### List Plans

```typescript
async function listPlans(): Promise<Plan[]> {
  const response = await fetch('https://api.paystack.co/plan', {
    headers: {
      Authorization: `Bearer ${process.env.PAYSTACK_SECRET_KEY}`,
    },
  });

  const result = await response.json();
  return result.data;
}
```

### Fetch Single Plan

```typescript
async function getPlan(planCodeOrId: string): Promise<Plan> {
  const response = await fetch(
    `https://api.paystack.co/plan/${encodeURIComponent(planCodeOrId)}`,
    {
      headers: {
        Authorization: `Bearer ${process.env.PAYSTACK_SECRET_KEY}`,
      },
    }
  );

  const result = await response.json();

  if (!result.status) {
    throw new Error(result.message);
  }

  return result.data;
}
```

### Update Plan

```typescript
interface UpdatePlanRequest {
  name?: string;
  amount?: number;
  interval?: string;
  description?: string;
  send_invoices?: boolean;
  send_sms?: boolean;
}

async function updatePlan(
  planCodeOrId: string,
  params: UpdatePlanRequest
): Promise<Plan> {
  const response = await fetch(
    `https://api.paystack.co/plan/${encodeURIComponent(planCodeOrId)}`,
    {
      method: 'PUT',
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
```

---

## Customers

Customers are automatically created on first payment, or you can create them explicitly.

### Create Customer

```
POST https://api.paystack.co/customer
```

```typescript
interface CreateCustomerRequest {
  email: string;
  first_name?: string;
  last_name?: string;
  phone?: string;
  metadata?: Record<string, unknown>;
}

interface Customer {
  id: number;
  email: string;
  customer_code: string;
  first_name: string | null;
  last_name: string | null;
  phone: string | null;
  metadata: Record<string, unknown> | null;
  integration: number;
  domain: 'test' | 'live';
  identified: boolean;
  identifications: unknown[] | null;
  created_at: string;
  updated_at: string;
}

async function createCustomer(params: CreateCustomerRequest): Promise<Customer> {
  const response = await fetch('https://api.paystack.co/customer', {
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
const customer = await createCustomer({
  email: 'user@example.com',
  first_name: 'John',
  last_name: 'Doe',
  metadata: {
    user_id: 'usr_123',
  },
});
```

### Fetch Customer

```typescript
async function getCustomer(emailOrCode: string): Promise<Customer> {
  const response = await fetch(
    `https://api.paystack.co/customer/${encodeURIComponent(emailOrCode)}`,
    {
      headers: {
        Authorization: `Bearer ${process.env.PAYSTACK_SECRET_KEY}`,
      },
    }
  );

  const result = await response.json();

  if (!result.status) {
    throw new Error(result.message);
  }

  return result.data;
}

// Can fetch by email or customer_code
const customer = await getCustomer('user@example.com');
// or
const customer = await getCustomer('CUS_xxx');
```

### Update Customer

```typescript
async function updateCustomer(
  customerCode: string,
  params: Partial<CreateCustomerRequest>
): Promise<Customer> {
  const response = await fetch(
    `https://api.paystack.co/customer/${encodeURIComponent(customerCode)}`,
    {
      method: 'PUT',
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
```

---

## Subscriptions

### Method 1: Initialize with Plan (Recommended)

The easiest way - subscription is created automatically after first payment.

```typescript
// Initialize transaction with plan parameter
const transaction = await initializeTransaction({
  email: 'user@example.com',
  amount: 500000, // Must match plan amount
  plan: 'PLN_xxx', // Plan code
  callback_url: 'https://yoursite.com/subscription/callback',
});

// User completes payment
// Subscription is automatically created
// You receive subscription.create webhook
```

### Method 2: Create Subscription Directly

For subscribing existing customers with saved cards.

```
POST https://api.paystack.co/subscription
```

```typescript
interface CreateSubscriptionRequest {
  customer: string; // Email or customer_code
  plan: string; // Plan code
  authorization?: string; // Authorization code (for specific card)
  start_date?: string; // ISO date for delayed start
}

interface Subscription {
  id: number;
  subscription_code: string;
  email_token: string;
  amount: number;
  cron_expression: string;
  next_payment_date: string;
  status: 'active' | 'non-renewing' | 'attention' | 'completed' | 'cancelled';
  plan: Plan;
  customer: Customer;
  authorization: {
    authorization_code: string;
    last4: string;
    card_type: string;
    bank: string;
    exp_month: string;
    exp_year: string;
  };
  created_at: string;
}

async function createSubscription(
  params: CreateSubscriptionRequest
): Promise<Subscription> {
  const response = await fetch('https://api.paystack.co/subscription', {
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

// Usage - Customer with existing authorization
const subscription = await createSubscription({
  customer: 'CUS_xxx',
  plan: 'PLN_xxx',
  authorization: 'AUTH_xxx', // Optional: use specific card
  start_date: '2024-02-01T00:00:00Z', // Optional: delayed start
});
```

### Fetch Subscription

```typescript
async function getSubscription(subscriptionCodeOrId: string): Promise<Subscription> {
  const response = await fetch(
    `https://api.paystack.co/subscription/${encodeURIComponent(subscriptionCodeOrId)}`,
    {
      headers: {
        Authorization: `Bearer ${process.env.PAYSTACK_SECRET_KEY}`,
      },
    }
  );

  const result = await response.json();

  if (!result.status) {
    throw new Error(result.message);
  }

  return result.data;
}
```

### List Subscriptions

```typescript
interface ListSubscriptionsParams {
  perPage?: number;
  page?: number;
  customer?: string; // Filter by customer
  plan?: string; // Filter by plan
}

async function listSubscriptions(
  params: ListSubscriptionsParams = {}
): Promise<Subscription[]> {
  const query = new URLSearchParams();
  if (params.perPage) query.set('perPage', params.perPage.toString());
  if (params.page) query.set('page', params.page.toString());
  if (params.customer) query.set('customer', params.customer);
  if (params.plan) query.set('plan', params.plan);

  const response = await fetch(
    `https://api.paystack.co/subscription?${query}`,
    {
      headers: {
        Authorization: `Bearer ${process.env.PAYSTACK_SECRET_KEY}`,
      },
    }
  );

  const result = await response.json();
  return result.data;
}

// Get all subscriptions for a customer
const subscriptions = await listSubscriptions({
  customer: 'CUS_xxx',
});
```

---

## Managing Subscriptions

### Enable Subscription

Re-activate a disabled subscription.

```
POST https://api.paystack.co/subscription/enable
```

```typescript
async function enableSubscription(
  code: string,
  token: string
): Promise<{ message: string }> {
  const response = await fetch('https://api.paystack.co/subscription/enable', {
    method: 'POST',
    headers: {
      Authorization: `Bearer ${process.env.PAYSTACK_SECRET_KEY}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      code, // subscription_code
      token, // email_token
    }),
  });

  const result = await response.json();

  if (!result.status) {
    throw new Error(result.message);
  }

  return result;
}
```

### Disable Subscription

Cancel a subscription (stops future charges).

```
POST https://api.paystack.co/subscription/disable
```

```typescript
async function disableSubscription(
  code: string,
  token: string
): Promise<{ message: string }> {
  const response = await fetch('https://api.paystack.co/subscription/disable', {
    method: 'POST',
    headers: {
      Authorization: `Bearer ${process.env.PAYSTACK_SECRET_KEY}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      code, // subscription_code
      token, // email_token
    }),
  });

  const result = await response.json();

  if (!result.status) {
    throw new Error(result.message);
  }

  return result;
}

// Usage
await disableSubscription('SUB_xxx', 'email_token_xxx');
```

### Generate Manage Link

Generate a link for customers to update their card or cancel subscription.

```
POST https://api.paystack.co/subscription/:code/manage/link
```

```typescript
async function generateManageLink(subscriptionCode: string): Promise<string> {
  const response = await fetch(
    `https://api.paystack.co/subscription/${subscriptionCode}/manage/link`,
    {
      method: 'POST',
      headers: {
        Authorization: `Bearer ${process.env.PAYSTACK_SECRET_KEY}`,
      },
    }
  );

  const result = await response.json();

  if (!result.status) {
    throw new Error(result.message);
  }

  return result.data.link;
}

// Usage
const link = await generateManageLink('SUB_xxx');
// Send this link to customer via email
// Link allows customer to update card or cancel
```

### Change Plan (Upgrade/Downgrade)

To change a subscription's plan:

1. Disable current subscription
2. Create new subscription with new plan

```typescript
async function changePlan(
  currentSubscriptionCode: string,
  emailToken: string,
  newPlanCode: string,
  customerCode: string
): Promise<Subscription> {
  // 1. Disable current subscription
  await disableSubscription(currentSubscriptionCode, emailToken);

  // 2. Create new subscription with new plan
  const newSubscription = await createSubscription({
    customer: customerCode,
    plan: newPlanCode,
    // Uses the same authorization by default
  });

  return newSubscription;
}
```

---

## Invoices

Invoices are created automatically 3 days before a subscription charge.

### List Invoices

```typescript
interface Invoice {
  id: number;
  domain: 'test' | 'live';
  amount: number;
  currency: string;
  due_date: string;
  status: 'pending' | 'success' | 'failed';
  paid: boolean;
  paid_at: string | null;
  description: string;
  authorization: {
    authorization_code: string;
    last4: string;
  };
  subscription: {
    id: number;
    subscription_code: string;
  };
  customer: Customer;
  transaction: number | null;
  created_at: string;
}

async function listInvoices(params: {
  perPage?: number;
  page?: number;
  customer?: string;
  status?: 'pending' | 'success' | 'failed';
  subscription?: string;
} = {}): Promise<Invoice[]> {
  const query = new URLSearchParams();
  Object.entries(params).forEach(([key, value]) => {
    if (value) query.set(key, value.toString());
  });

  const response = await fetch(
    `https://api.paystack.co/invoice?${query}`,
    {
      headers: {
        Authorization: `Bearer ${process.env.PAYSTACK_SECRET_KEY}`,
      },
    }
  );

  const result = await response.json();
  return result.data;
}

// Get pending invoices for a customer
const pendingInvoices = await listInvoices({
  customer: 'CUS_xxx',
  status: 'pending',
});
```

### Invoice Lifecycle

```
┌─────────────────┐
│ invoice.create  │ ← 3 days before due date
│ status: pending │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Auto-charge on  │ ← On due date
│ due date        │
└────────┬────────┘
         │
    ┌────┴────┐
    ▼         ▼
┌───────┐ ┌────────────────────┐
│Success│ │ invoice.payment_   │
│       │ │ failed (attempts)  │
└───┬───┘ └────────┬───────────┘
    │              │
    ▼              ▼
┌───────────┐ ┌───────────────┐
│ charge.   │ │ subscription  │
│ success   │ │ .disable      │
│           │ │ (after retries)│
└───────────┘ └───────────────┘
```

---

## Subscription Status Reference

| Status | Description | Action |
|--------|-------------|--------|
| `active` | Subscription is active and will renew | Normal state |
| `non-renewing` | Active but won't renew (user cancelled) | Access until end of period |
| `attention` | Payment issue, needs card update | Notify user to update card |
| `completed` | Reached invoice_limit | Create new subscription if needed |
| `cancelled` | Fully cancelled | Revoke access |

---

## Complete Subscription Example

```typescript
// 1. Setup: Create plans (usually one-time via Dashboard)
const basicPlan = await createPlan({
  name: 'Basic',
  amount: 299900, // ₦2,999
  interval: 'monthly',
});

const proPlan = await createPlan({
  name: 'Pro',
  amount: 999900, // ₦9,999
  interval: 'monthly',
});

// 2. Subscribe user
async function subscribeUser(email: string, planCode: string) {
  const transaction = await initializeTransaction({
    email,
    amount: (await getPlan(planCode)).amount,
    plan: planCode,
    callback_url: `${process.env.FRONTEND_URL}/subscription/callback`,
    metadata: {
      user_id: 'usr_123',
    },
  });

  return transaction.authorization_url;
}

// 3. Handle webhook for subscription.create
async function handleSubscriptionCreate(data: Record<string, unknown>) {
  const subscriptionCode = data.subscription_code as string;
  const customerEmail = (data.customer as { email: string }).email;
  const planCode = (data.plan as { plan_code: string }).plan_code;

  await db.users.update({
    where: { email: customerEmail },
    data: {
      subscriptionCode,
      subscriptionStatus: 'active',
      planCode,
      subscriptionStartedAt: new Date(),
    },
  });
}

// 4. Handle invoice.payment_failed
async function handleInvoicePaymentFailed(data: Record<string, unknown>) {
  const customerEmail = (data.customer as { email: string }).email;
  const attempt = data.attempt as number;

  // Notify user
  await sendEmail({
    to: customerEmail,
    subject: 'Payment failed - Please update your card',
    template: 'payment-failed',
    data: { attempt },
  });

  // After multiple failures, Paystack cancels the subscription
  // You'll receive subscription.disable webhook
}

// 5. Provide management link to users
async function getManageLink(userId: string): Promise<string> {
  const user = await db.users.findUnique({ where: { id: userId } });

  if (!user?.subscriptionCode) {
    throw new Error('No active subscription');
  }

  return generateManageLink(user.subscriptionCode);
}
```
