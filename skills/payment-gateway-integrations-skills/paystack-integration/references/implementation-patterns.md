> **When to read:** When translating patterns to another language or framework.
> **What problem it solves:** Provides framework-agnostic pseudo-code for init/verify/webhook flows.
> **When to skip:** If you only need concrete framework examples (see nextjs/express files).
> **Prerequisites:** Read AGENT_EXECUTION_SPEC.md first.

# Implementation Patterns (Framework-Agnostic Pseudocode)

Initialize Flow

```
function initializePayment(orderId, email, amountSmallestUnit):
  // STEP 1: Call Paystack FIRST — do NOT provide reference; let Paystack generate it
  response = http.post('/transaction/initialize', {
    email: email,
    amount: amountSmallestUnit
    // reference is intentionally omitted — Paystack generates one
  })

  if not response.success:
    // handle retry policies externally
    throw Error('initialize failed')

  // STEP 2: Extract Paystack's reference from response
  paystackReference = response.data.reference

  // STEP 3: Persist order with Paystack's reference AFTER successful initialization
  // CRITICAL: Use paystackReference, not a locally-generated one
  // This reference will appear in webhooks and must match your database
  db.save(orderId, { reference: paystackReference, amount: amountSmallestUnit, status: 'pending' })

  return { authorization_url: response.data.authorization_url, reference: paystackReference }
```

⚠️ **WHY THIS ORDER**: By calling Paystack before saving to DB, you avoid orphan records if the API call fails. The reference from Paystack is the source of truth for verification and webhooks.

Verify Flow

```
function verifyPayment(reference):
  // Fetch expected amount from DB (in smallest unit)
  order = db.findByReference(reference)
  if not order:
    throw Error('missing order')
  if order.status == 'paid':
    return order // idempotent exit

  // Call Paystack verify API
  response = http.get('/transaction/verify/' + encode(reference))
  if not response.success or response.data.status != 'success':
    throw Error('payment not successful')

  if response.data.amount != order.amount:
    throw Error('amount mismatch')

  // Mark order paid atomically
  db.update(order.id, { status: 'paid', paid_at: now() })
  return db.find(order.id)
```

Webhook Handler Structure

```
function webhookHandler(rawBody, signatureHeader):
  if not verifySignature(rawBody, signatureHeader, secret):
    return http.respond(401, 'invalid signature')

  // Respond 200 ASAP (ack) and process asynchronously if possible
  event = parseJson(rawBody)
  enqueueJob('process-webhook', event)
  return http.respond(200, 'OK')
```

Signature Verification Logic

```
function verifySignature(rawBody, signatureHeader, secret):
  computed = hmac_sha512(secret, rawBody)
  return timingSafeEqual(hex(computed), signatureHeader)
```

Purpose: Translate these patterns into Python, Go, PHP, Ruby, etc., without framework-specific constructs.
