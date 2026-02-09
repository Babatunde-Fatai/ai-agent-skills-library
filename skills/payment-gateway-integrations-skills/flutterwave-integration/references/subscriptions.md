> **When to read:** When implementing subscription plans, recurring billing, or payment plans.
> **What problem it solves:** Guides plan creation, subscription setup, and recurring payment management.
> **When to skip:** If you only need one-time payments.
> **Prerequisites:** Read AGENT_EXECUTION_SPEC.md first.

# Subscriptions & Recurring Billing - Complete Guide

This guide covers subscription plans, payment plans, and recurring billing with Flutterwave.

## Table of Contents

1. [Overview](#overview)
2. [Payment Plans](#payment-plans)
3. [Subscriptions](#subscriptions)
4. [Managing Subscriptions](#managing-subscriptions)
5. [Webhook Handling](#webhook-handling)
6. [Complete Example](#complete-example)

---

## Overview

Flutterwave supports two approaches for recurring payments:

1. **Payment Plans** - Define billing amount and interval, attach to transactions
2. **Subscriptions** - Create subscriptions that automatically charge customers

### Subscription Flow

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        SUBSCRIPTION FLOW                                  │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  SETUP (One-time)                                                        │
│  ┌─────────────────┐                                                     │
│  │  Create Plan    │ → plan_id                                           │
│  │  (Dashboard/API)│   Define: name, amount, interval                    │
│  └─────────────────┘                                                     │
│                                                                          │
│  SUBSCRIBE (Per customer)                                                │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐      │
│  │ Initialize with │───▶│ Customer pays   │───▶│ Subscription    │      │
│  │ payment_plan    │    │ first time      │    │ auto-created    │      │
│  └─────────────────┘    └─────────────────┘    └─────────────────┘      │
│                                                                          │
│  LIFECYCLE (Automatic)                                                   │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐      │
│  │ Reminder sent   │───▶│ Auto-charge     │───▶│ charge.completed│      │
│  │ before due date │    │ (on due date)   │    │ OR failed       │      │
│  └─────────────────┘    └─────────────────┘    └─────────────────┘      │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## Payment Plans

Payment plans define the billing structure for subscriptions.

### Create Plan via Dashboard

1. Go to **Flutterwave Dashboard** → **Payments** → **Payment Plans**
2. Click **Create Plan**
3. Fill in: Name, Amount, Interval, Duration
4. Save and note the `plan_id`

### Create Plan via API

```
POST https://api.flutterwave.com/v3/payment-plans
```

### Request Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `name` | string | Yes | Plan name |
| `amount` | number | Yes | Amount per interval |
| `interval` | string | Yes | `daily`, `weekly`, `monthly`, `quarterly`, `yearly` |
| `duration` | number | No | Number of intervals (0 = unlimited) |
| `currency` | string | No | Currency code (default: NGN) |

### Implementation

```typescript
interface CreatePlanRequest {
  name: string;
  amount: number;  // In smallest unit
  interval: 'daily' | 'weekly' | 'monthly' | 'quarterly' | 'yearly';
  duration?: number;  // 0 = unlimited
  currency?: string;
}

interface Plan {
  id: number;
  name: string;
  amount: number;
  interval: string;
  duration: number;
  status: string;
  currency: string;
  plan_token: string;
  created_at: string;
}

async function createPlan(params: CreatePlanRequest): Promise<Plan> {
  const response = await fetch(`${FLW_BASE_URL}/payment-plans`, {
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
const plan = await createPlan({
  name: 'Pro Monthly',
  amount: toSmallestUnit(5000),  // 5,000 NGN per month
  interval: 'monthly',
  duration: 0,  // Unlimited
  currency: 'NGN',
});

console.log(plan.id);  // Use this for subscriptions
```

### List Plans

```typescript
async function listPlans(): Promise<Plan[]> {
  const response = await fetch(`${FLW_BASE_URL}/payment-plans`, {
    headers: getHeaders(),
  });

  const result = await response.json();
  return result.data;
}
```

### Get Single Plan

```typescript
async function getPlan(planId: number): Promise<Plan> {
  const response = await fetch(
    `${FLW_BASE_URL}/payment-plans/${planId}`,
    { headers: getHeaders() }
  );

  const result = await response.json();

  if (result.status !== 'success') {
    throw new Error(result.message);
  }

  return result.data;
}
```

### Update Plan

```typescript
async function updatePlan(
  planId: number,
  params: Partial<CreatePlanRequest>
): Promise<Plan> {
  const response = await fetch(
    `${FLW_BASE_URL}/payment-plans/${planId}`,
    {
      method: 'PUT',
      headers: getHeaders(),
      body: JSON.stringify(params),
    }
  );

  const result = await response.json();

  if (result.status !== 'success') {
    throw new Error(result.message);
  }

  return result.data;
}
```

### Cancel Plan

```typescript
async function cancelPlan(planId: number): Promise<void> {
  const response = await fetch(
    `${FLW_BASE_URL}/payment-plans/${planId}/cancel`,
    {
      method: 'PUT',
      headers: getHeaders(),
    }
  );

  const result = await response.json();

  if (result.status !== 'success') {
    throw new Error(result.message);
  }
}
```

---

## Subscriptions

### Method 1: Initialize with Payment Plan (Recommended)

The easiest approach - subscription is created automatically after first payment.

```typescript
// Initialize transaction with payment_plan parameter
const txRef = generateTxRef('SUB');

const payment = await initializePayment({
  tx_ref: txRef,
  amount: toSmallestUnit(5000),  // Must match plan amount
  currency: 'NGN',
  redirect_url: 'https://yoursite.com/subscription/callback',
  payment_plan: planId,  // Your plan ID
  customer: {
    email: 'user@example.com',
    name: 'John Doe',
  },
});

// User completes payment
// Subscription is automatically created
// You receive charge.completed webhook with subscription info
```

### Get Subscriptions for a Plan

```typescript
interface Subscription {
  id: number;
  amount: number;
  customer: {
    id: number;
    customer_email: string;
  };
  plan: number;
  status: 'active' | 'cancelled';
  created_at: string;
}

async function getPlanSubscriptions(planId: number): Promise<Subscription[]> {
  const response = await fetch(
    `${FLW_BASE_URL}/subscriptions?plan=${planId}`,
    { headers: getHeaders() }
  );

  const result = await response.json();
  return result.data;
}
```

### Get All Subscriptions

```typescript
async function getAllSubscriptions(): Promise<Subscription[]> {
  const response = await fetch(`${FLW_BASE_URL}/subscriptions`, {
    headers: getHeaders(),
  });

  const result = await response.json();
  return result.data;
}
```

### Activate Subscription

Re-enable a cancelled subscription.

```typescript
async function activateSubscription(subscriptionId: number): Promise<Subscription> {
  const response = await fetch(
    `${FLW_BASE_URL}/subscriptions/${subscriptionId}/activate`,
    {
      method: 'PUT',
      headers: getHeaders(),
    }
  );

  const result = await response.json();

  if (result.status !== 'success') {
    throw new Error(result.message);
  }

  return result.data;
}
```

### Cancel Subscription

```typescript
async function cancelSubscription(subscriptionId: number): Promise<Subscription> {
  const response = await fetch(
    `${FLW_BASE_URL}/subscriptions/${subscriptionId}/cancel`,
    {
      method: 'PUT',
      headers: getHeaders(),
    }
  );

  const result = await response.json();

  if (result.status !== 'success') {
    throw new Error(result.message);
  }

  return result.data;
}
```

---

## Managing Subscriptions

### Check Subscription Status

```typescript
async function getSubscriptionStatus(
  customerEmail: string,
  planId: number
): Promise<Subscription | null> {
  const subscriptions = await getPlanSubscriptions(planId);

  return subscriptions.find(
    sub => sub.customer.customer_email === customerEmail && sub.status === 'active'
  ) || null;
}

// Usage
const subscription = await getSubscriptionStatus('user@example.com', planId);

if (subscription) {
  console.log('User has active subscription');
} else {
  console.log('User needs to subscribe');
}
```

### Change Plan (Upgrade/Downgrade)

To change a subscription's plan:

1. Cancel current subscription
2. Create new payment with new plan

```typescript
async function changePlan(
  currentSubscriptionId: number,
  customerEmail: string,
  newPlanId: number
): Promise<string> {
  // 1. Cancel current subscription
  await cancelSubscription(currentSubscriptionId);

  // 2. Get new plan details
  const newPlan = await getPlan(newPlanId);

  // 3. Initialize new subscription payment
  const txRef = generateTxRef('UPG');

  const payment = await initializePayment({
    tx_ref: txRef,
    amount: newPlan.amount,
    currency: newPlan.currency,
    redirect_url: 'https://yoursite.com/subscription/callback',
    payment_plan: newPlanId,
    customer: {
      email: customerEmail,
    },
  });

  return payment.link;
}
```

### Track Subscription in Database

```typescript
// Example database schema
interface UserSubscription {
  id: string;
  userId: string;
  planId: number;
  subscriptionId: number;
  status: 'active' | 'cancelled' | 'expired';
  currentPeriodStart: Date;
  currentPeriodEnd: Date;
  createdAt: Date;
  updatedAt: Date;
}

// When subscription is created (from webhook)
async function handleNewSubscription(webhookData: {
  customer: { customer_email: string };
  plan: number;
  id: number;
}) {
  const user = await db.users.findUnique({
    where: { email: webhookData.customer.customer_email },
  });

  if (!user) return;

  // Get plan interval to calculate period end
  const plan = await getPlan(webhookData.plan);
  const periodEnd = calculatePeriodEnd(plan.interval);

  await db.subscriptions.create({
    data: {
      userId: user.id,
      planId: webhookData.plan,
      subscriptionId: webhookData.id,
      status: 'active',
      currentPeriodStart: new Date(),
      currentPeriodEnd: periodEnd,
    },
  });
}

function calculatePeriodEnd(interval: string): Date {
  const now = new Date();
  switch (interval) {
    case 'daily': return new Date(now.setDate(now.getDate() + 1));
    case 'weekly': return new Date(now.setDate(now.getDate() + 7));
    case 'monthly': return new Date(now.setMonth(now.getMonth() + 1));
    case 'quarterly': return new Date(now.setMonth(now.getMonth() + 3));
    case 'yearly': return new Date(now.setFullYear(now.getFullYear() + 1));
    default: return new Date(now.setMonth(now.getMonth() + 1));
  }
}
```

---

## Webhook Handling

### Subscription Events

```typescript
interface SubscriptionWebhookEvent {
  event: 'subscription.cancelled';
  data: {
    id: number;
    amount: number;
    customer: {
      id: number;
      customer_email: string;
    };
    plan: number;
    status: 'cancelled';
    created_at: string;
  };
}

async function handleSubscriptionCancelled(data: SubscriptionWebhookEvent['data']) {
  const customerEmail = data.customer.customer_email;
  const subscriptionId = data.id;

  // Update subscription status
  // await db.subscriptions.update({
  //   where: { subscriptionId },
  //   data: { status: 'cancelled' },
  // });

  // Optionally revoke access
  // await revokeUserAccess(customerEmail);

  // Send cancellation email
  // await sendCancellationEmail(customerEmail);

  console.log(`Subscription ${subscriptionId} cancelled for ${customerEmail}`);
}
```

### Recurring Charge Events

When a subscription renews:

```typescript
interface RecurringChargeEvent {
  event: 'charge.completed';
  data: {
    id: number;
    tx_ref: string;
    flw_ref: string;
    amount: number;
    currency: string;
    status: 'successful';
    payment_type: string;
    customer: {
      email: string;
    };
    // For subscriptions, this will be populated
    plan?: number;
  };
}

async function handleRecurringCharge(data: RecurringChargeEvent['data']) {
  // Check if this is a subscription payment
  if (!data.plan) return;

  const customerEmail = data.customer.email;
  const planId = data.plan;

  // Update subscription period
  // await db.subscriptions.update({
  //   where: {
  //     userId_planId: { userId, planId },
  //   },
  //   data: {
  //     currentPeriodStart: new Date(),
  //     currentPeriodEnd: calculatePeriodEnd(planInterval),
  //   },
  // });

  console.log(`Subscription renewed for ${customerEmail}`);
}
```

---

## Complete Example

### Subscription Checkout Flow

```typescript
// 1. Create plans (usually one-time via Dashboard or migration)
const basicPlan = await createPlan({
  name: 'Basic',
  amount: toSmallestUnit(2999),  // 2,999 NGN
  interval: 'monthly',
  currency: 'NGN',
});

const proPlan = await createPlan({
  name: 'Pro',
  amount: toSmallestUnit(9999),  // 9,999 NGN
  interval: 'monthly',
  currency: 'NGN',
});

// 2. Subscribe user endpoint
async function subscribeUser(
  email: string,
  planId: number
): Promise<string> {
  const plan = await getPlan(planId);
  const txRef = generateTxRef('SUB');

  // Store pending subscription
  // await db.pendingSubscriptions.create({
  //   data: { txRef, email, planId, status: 'pending' },
  // });

  const payment = await initializePayment({
    tx_ref: txRef,
    amount: plan.amount,
    currency: plan.currency,
    redirect_url: `${process.env.FRONTEND_URL}/subscription/callback`,
    payment_plan: planId,
    customer: {
      email,
    },
    customizations: {
      title: 'Subscribe to ' + plan.name,
      description: `${plan.interval} subscription`,
    },
  });

  return payment.link;
}

// 3. Handle webhook for subscription activation
async function handleChargeCompleted(data: {
  tx_ref: string;
  amount: number;
  status: string;
  customer: { email: string };
  plan?: number;
}) {
  if (data.status !== 'successful') return;

  // If this is a subscription payment
  if (data.plan) {
    const email = data.customer.email;

    // Activate user subscription in your system
    // await db.users.update({
    //   where: { email },
    //   data: {
    //     subscriptionPlanId: data.plan,
    //     subscriptionStatus: 'active',
    //     subscriptionStartedAt: new Date(),
    //   },
    // });

    console.log(`Subscription activated for ${email}`);
  }
}

// 4. Check subscription status (for access control)
async function hasActiveSubscription(userId: string): Promise<boolean> {
  // const user = await db.users.findUnique({ where: { id: userId } });
  // return user?.subscriptionStatus === 'active';
  return true;
}

// 5. Provide management endpoint
async function cancelUserSubscription(userId: string): Promise<void> {
  // const user = await db.users.findUnique({ where: { id: userId } });
  // if (!user?.subscriptionId) throw new Error('No subscription found');

  // const subscriptions = await getAllSubscriptions();
  // const userSub = subscriptions.find(
  //   s => s.customer.customer_email === user.email && s.status === 'active'
  // );

  // if (userSub) {
  //   await cancelSubscription(userSub.id);
  // }

  // Update local status
  // await db.users.update({
  //   where: { id: userId },
  //   data: { subscriptionStatus: 'cancelled' },
  // });
}
```

---

## Subscription Status Reference

| Status | Description | Action |
|--------|-------------|--------|
| `active` | Subscription is active | User has access |
| `cancelled` | Subscription cancelled | Revoke access at period end |

---

## Best Practices

1. **Store subscription info locally** - Don't rely solely on Flutterwave API calls
2. **Use webhooks** - Don't poll for subscription status
3. **Handle failed payments** - Implement retry logic or grace period
4. **Send reminders** - Notify users before renewal
5. **Prorate changes** - Handle upgrades/downgrades fairly
