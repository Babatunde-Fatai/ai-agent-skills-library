> **When to read:** First thing—read this before coding. It is the agent execution contract.
> **What problem it solves:** Defines non-negotiable safety invariants and role separation.
> **When to skip:** Do NOT skip; only skip when making tiny docs edits that don't affect code.
> **Prerequisites:** None — this is the primary prerequisite for implementation.

# Flutterwave Agent Execution Spec

This file is the execution contract for coding agents integrating Flutterwave.

Do not start implementation from examples. Follow this contract first.

Examples in this skill are reference material. This file defines the rules you must obey.

---
## Role Selection (Read First)

This skill defines two roles. Before writing any code, determine which one you are implementing.

### Frontend Role — Payment Collection Only
You are in this role if working on UI, popup, redirect, or browser code.

You MUST implement:
- Redirect to Flutterwave hosted payment page OR
- Flutterwave inline modal using JavaScript SDK
- Sending the `tx_ref` to the backend after payment

You MUST NOT:
- Call Flutterwave verify API
- Use FLW_SECRET_KEY
- Handle webhooks
- Mark payments as successful

---

### Backend Role — Payment Authority
You are in this role if working on API routes, database logic, verification, or webhooks.

You MUST implement:
- Generate unique `tx_ref` BEFORE calling initialize
- Initialize transaction with `tx_ref`, `amount`, `currency`, `redirect_url`
- Store order with `tx_ref` and status=pending BEFORE redirecting user
- Verify transaction using `tx_ref` or `id` after payment
- Compare verified amount with DB amount (in smallest unit)
- Webhook signature verification (verif-hash OR HMAC-SHA256)
- Idempotent fulfillment

You MUST NOT:
- Use Flutterwave public key for API calls
- Implement popup or redirect UI logic

___

## Payment Invariants (Non-Negotiable Rules)

1. NEVER trust client-side success callbacks.
2. NEVER fulfill an order after calling initializeTransaction.
3. ALWAYS generate unique `tx_ref` and store in database with status=pending BEFORE redirecting user.
4. ALWAYS convert currency to smallest unit (kobo/pesewas/cents) before calling Flutterwave APIs.
5. ALWAYS verify the transaction on your backend using `tx_ref` or transaction `id`.
6. ALWAYS compare the verified amount against the expected database amount.
7. Webhook events are a source of truth and must be handled idempotently.
8. Webhook signature verification REQUIRES raw request body (for HMAC method).
9. NEVER parse webhook JSON before verifying the signature (for HMAC method).
10. If an order is already marked paid, webhook or verify handlers must exit immediately.

Failure to follow any rule above creates fraud risk.

---

## Required Roles (Framework Agnostic)

You must implement these roles regardless of framework or language.

### Minimal Data Contract (Do this before writing any code)

Your system MUST be able to persist and query, at minimum, the following fields before and after payment:

- tx_ref (unique)
- amount in smallest unit
- currency
- status (pending, paid, failed, etc.)
- transaction_id (after verification)

These may exist in any schema or table structure in your system.

### Role: Reference Generator
- Generate unique `tx_ref` using timestamp + random string
- Format: `FLW_{timestamp}_{random}` or similar unique pattern
- This reference links your system to Flutterwave's transaction

### Role: Payment Initializer
- Accepts email, amount, currency, redirect_url, metadata.
- Converts amount to smallest unit (kobo for NGN, pesewas for GHS, cents for others).
- Generates unique `tx_ref`.
- Stores order with status = pending, `tx_ref`, and amount (in smallest unit).
- Calls Flutterwave `/v3/payments` endpoint.
- Returns `link` (redirect URL) to frontend.

### Role: Transaction Verifier
- Accepts `tx_ref` or transaction `id`.
- Fetches expected amount from database.
- Calls Flutterwave `/v3/transactions/:id/verify` or `/v3/transactions/verify_by_reference?tx_ref=xxx`.
- Confirms:
  - status === 'successful'
  - amount === expected amount
  - currency === expected currency
- Handles `success-pending-validation` status appropriately (poll or wait for webhook).
- Marks order as paid.

### Role: Webhook Verifier (Two Methods): (HMAC is preferred. Use verif-hash only if raw body access is impossible)

**Method 1: HMAC-SHA256 (flutterwave-signature header)**
- Capture raw request body BEFORE JSON parsing.
- Compute HMAC-SHA256 of raw body using `FLW_SECRET_HASH`.
- Base64-encode the result.
- Compare to `flutterwave-signature` header using timing-safe comparison.
- If mismatch, reject with 401.

**Method 2: Simple Hash Comparison (verif-hash header)**
- Extract `verif-hash` header from request.
- Compare directly to your `FLW_SECRET_HASH` environment variable.
- If mismatch, reject with 401.

### Role: Webhook Handler (Idempotent)
- On `charge.completed`:
  - If order already paid, exit.
  - Otherwise run the same logic as Transaction Verifier.
- On subscription events, update subscription state.
- CRITICAL: Return HTTP 200 OK immediately after signature verification, BEFORE running complex fulfillment logic.
- For MVP, you may process fulfillment asynchronously after sending the response. For production scale, use a queue or background worker.


---

## Canonical Payment Flow (Follow Exactly)

When user wants to pay:

1. Generate unique `tx_ref` (e.g., `FLW_LK5J2M8_A1B2C3D4`).
2. Store order in DB:
   - tx_ref
   - amountInSmallestUnit
   - currency
   - status = pending
3. Call Flutterwave `/v3/payments` with `tx_ref`, amount, currency, redirect_url.
4. Flutterwave returns `data.link` (hosted payment URL).
5. Return `link` to frontend for redirect.

⚠️ **CRITICAL DIFFERENCE FROM PAYSTACK**: With Flutterwave, YOU generate the `tx_ref` before calling the API. Store it in your DB first so webhooks can find the order.

When user returns from Flutterwave OR frontend reports success:

6. Call verifyTransaction(tx_ref) or verifyTransaction(transaction_id).
7. Fetch expectedAmount from DB.
8. Compare amounts and currency.
9. Check status is 'successful' (not 'pending' or 'success-pending-validation').
10. If valid, mark order paid.

When webhook arrives:

11. Verify signature using verif-hash OR HMAC method.
12. If event is `charge.completed`, repeat steps 6 to 10.
13. If order already paid, exit.

---

## Transaction Status Values

| Status | Meaning | Action |
|--------|---------|--------|
| `successful` | Payment completed | Fulfill order |
| `pending` | Awaiting completion | Poll or wait for webhook |
| `success-pending-validation` | ACH/bank transfer pending | Wait for webhook confirmation |
| `failed` | Payment failed | Notify user |

---

## Idempotency Rule

All verify and webhook handlers must begin with:

```
if order.status == 'paid':
  exit
```

This prevents double fulfillment.

---

## Amount Conversion Rule

All amounts sent to Flutterwave MUST be in smallest currency unit:

| Currency | Smallest Unit | Multiplier |
|----------|---------------|------------|
| NGN | kobo | 100 |
| GHS | pesewas | 100 |
| KES | cents | 100 |
| ZAR | cents | 100 |
| XOF | francs | 100 |
| USD | cents | 100 |

All amounts stored in DB for comparison MUST be in smallest unit.

Never compare naira to kobo.

Always ROUND when converting to smallest unit. Never truncate. Example: 100.5 NGN → 10050 kobo.

---

## Webhook Raw Body Rule (For HMAC Method)

Many frameworks auto-parse JSON. This breaks HMAC signature verification.

You must capture the raw request buffer BEFORE JSON parsing.

If you cannot access raw body, use the simpler `verif-hash` method instead.

---

## Anti-Patterns (Do NOT Do These)

❌ Fulfill order after initializeTransaction
❌ Trust popup success callback
❌ Parse webhook body before verifying signature (HMAC method)
❌ Use amount in naira for Flutterwave API calls
❌ Skip database lookup during verification
❌ Mark order paid inside webhook without verifying amount
❌ Forget to handle `success-pending-validation` status
❌ Use same `tx_ref` for different transactions

---

## Agent Failure Triggers (If you are doing any of these, your implementation is incorrect)

- If you mark an order as paid inside the payment initialization route, you are wrong.

- If you do not store tx_ref in your database before calling Flutterwave, you are wrong.

- If you trust the frontend success callback without backend verification, you are wrong.

- If you compare the verified amount to the original amount in naira/cedi/shilling instead of smallest unit, you are wrong.

- If your webhook handler does not check whether the order is already paid before processing, you are wrong.

- If you parse the webhook JSON body before verifying the HMAC signature, you are wrong.

- If you use the Flutterwave public key in any backend API call, you are wrong.

- If your system cannot find an order by tx_ref when a webhook arrives, you are wrong.

- If you fulfill an order when the transaction status is `pending` or `success-pending-validation`, you are wrong.

---

## What Examples Are For

All example implementations in this skill show how to implement these roles in specific frameworks.

They are references, not the protocol.

Follow this spec first, then map examples to your framework.
