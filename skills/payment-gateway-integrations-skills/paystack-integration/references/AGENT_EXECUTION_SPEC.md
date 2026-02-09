> **When to read:** First thing—read this before coding. It is the agent execution contract.
> **What problem it solves:** Defines non-negotiable safety invariants and role separation.
> **When to skip:** Do NOT skip; only skip when making tiny docs edits that don't affect code.
> **Prerequisites:** None — this is the primary prerequisite for implementation.

# Paystack Agent Execution Spec

This file is the execution contract for coding agents integrating Paystack.

Do not start implementation from examples. Follow this contract first.

Examples in this skill are reference material. This file defines the rules you must obey.

---
## Role Selection (Read First)

This skill defines two roles. Before writing any code, determine which one you are implementing.

### Frontend Role — Payment Collection Only
You are in this role if working on UI, popup, redirect, or browser code.

You MUST implement:
- Redirect to `authorization_url` OR
- Paystack Inline.js popup using `access_code`
- Sending the `reference` to the backend after payment

You MUST NOT:
- Call Paystack verify API
- Use PAYSTACK_SECRET_KEY
- Handle webhooks
- Mark payments as successful

---

### Backend Role — Payment Authority
You are in this role if working on API routes, database logic, verification, or webhooks.

You MUST implement:
- Initialize transaction (let Paystack generate reference)
- Store order with Paystack's reference AFTER successful initialization
- Verify transaction using reference
- Compare verified amount with DB amount (kobo)
- Webhook signature verification using raw body
- Idempotent fulfillment

You MUST NOT:
- Use Paystack public key
- Implement popup or redirect UI logic

___

## Payment Invariants (Non-Negotiable Rules)

1. NEVER trust client-side success callbacks.
2. NEVER fulfill an order after calling initializeTransaction.
3. ALWAYS store the transaction reference in your database BEFORE redirecting the user.
4. ALWAYS convert currency to kobo before calling Paystack APIs.
5. ALWAYS verify the transaction on your backend using the reference.
6. ALWAYS compare the verified amount against the expected database amount.
7. Webhook events are a source of truth and must be handled idempotently.
8. Webhook signature verification REQUIRES the raw request body.
9. NEVER parse webhook JSON before verifying the signature.
10. If an order is already marked paid, webhook or verify handlers must exit immediately.
11. On network failure during initialization where response is ambiguous (timeout, connection reset), query Paystack to check if the transaction exists before retrying. If retrying a known reference, pass it explicitly.

Failure to follow any rule above creates fraud risk.

---

## Required Roles (Framework Agnostic)

You must implement these roles regardless of framework or language.

### Role: Payment Initializer
- Accepts email, amount, metadata.
- Converts amount to kobo.
- Calls Paystack `/transaction/initialize` (without reference — let Paystack generate it).
- Extracts reference from Paystack response.
- Stores order with status = pending and **Paystack's reference**.
- Returns authorization_url, access_code, and reference.

### Role: Transaction Verifier
- Accepts reference.
- Fetches expected amount from database.
- Calls Paystack `/transaction/verify/:reference`.
- Confirms:
  - status === 'success'
  - amount === expected amount
- Marks order as paid.

### Role: Webhook Verifier
- Captures raw body.
- Verifies `x-paystack-signature` using raw body.
- Parses JSON ONLY after verification.

### Role: Webhook Handler (Idempotent)
- On `charge.success`:
  - If order already paid, exit.
  - Otherwise run the same logic as Transaction Verifier.
- On subscription events, update subscription state.
- CRITICAL: Return HTTP 200 OK immediately after signature verification, BEFORE running complex fulfillment logic.

---

## Canonical Payment Flow (Follow Exactly)

When user wants to pay:

1. Call initializeTransaction (omit reference parameter — Paystack generates one).
2. Extract reference from Paystack response.
3. Store order in DB with **Paystack's reference**:
   - reference (from Paystack response — CRITICAL)
   - amountInKobo
   - status = pending
4. Return authorization_url, access_code, and reference to frontend.

⚠️ **WHY THIS ORDER**: If you save to DB before calling Paystack and the API call fails, you create orphan records. By calling Paystack first, you only persist successful initializations.

⚠️ **REFERENCE CONSISTENCY**: The reference stored in your DB MUST match the one returned by Paystack. This same reference will appear in webhooks and verification responses.

When user returns from Paystack OR frontend popup reports success:

5. Call verifyTransaction(reference).
6. Fetch expectedAmount from DB.
7. Compare amounts.
8. If valid, mark order paid.

When webhook arrives:

9. Verify signature using raw body.
10. If event is `charge.success`, repeat steps 5 to 8.
11. If order already paid, exit.

---

## Idempotency Rule

All verify and webhook handlers must begin with:

```
if order.status == 'paid':
  exit
```

This prevents double fulfillment.

---

## Kobo Rule

All amounts sent to Paystack MUST be in kobo.

All amounts stored in DB for comparison MUST be in kobo.

Never compare naira to kobo.

---

## Webhook Raw Body Rule (Critical)

Many frameworks auto-parse JSON. This breaks Paystack signature verification.

You must capture the raw request buffer BEFORE JSON parsing.

If you cannot access raw body, webhook verification will fail.

---

## Anti-Patterns (Do NOT Do These)

❌ Fulfill order after initializeTransaction  
❌ Trust popup success callback  
❌ Parse webhook body before verifying signature  
❌ Use amount in naira for Paystack API calls  
❌ Skip database lookup during verification  
❌ Mark order paid inside webhook without verifying amount  

---

## What Examples Are For

All example implementations in this skill show how to implement these roles in specific frameworks.

They are references, not the protocol.

Follow this spec first, then map examples to your framework.