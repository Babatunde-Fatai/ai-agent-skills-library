> **When to read:** When implementing in a framework not covered by specific guides, or when you need framework-agnostic patterns.
> **What problem it solves:** Provides pseudocode and patterns that can be adapted to any language/framework.
> **When to skip:** If your framework has a dedicated implementation guide.
> **Prerequisites:** Read AGENT_EXECUTION_SPEC.md first.

# Implementation Patterns - Framework Agnostic

This guide provides pseudocode and patterns that can be adapted to any programming language or framework.

## Table of Contents

1. [Core Payment Flow](#core-payment-flow)
2. [Data Models](#data-models)
3. [Service Layer](#service-layer)
4. [Webhook Processing](#webhook-processing)
5. [Error Handling](#error-handling)
6. [State Machine](#state-machine)

---

## Core Payment Flow

### Canonical Payment Sequence

```
FUNCTION initiate_payment(customer_email, amount, currency):
    // STEP 1: Generate unique reference
    tx_ref = generate_unique_reference("FLW")

    // STEP 2: Store order in database FIRST
    order = database.create_order({
        tx_ref: tx_ref,
        email: customer_email,
        amount: to_smallest_unit(amount),
        currency: currency,
        status: "pending",
        created_at: now()
    })

    // STEP 3: Call Flutterwave API
    response = flutterwave.initialize_payment({
        tx_ref: tx_ref,
        amount: to_smallest_unit(amount),
        currency: currency,
        redirect_url: config.callback_url,
        customer: { email: customer_email }
    })

    IF response.status != "success":
        // Mark order as failed
        database.update_order(order.id, { status: "init_failed" })
        THROW PaymentInitError(response.message)

    // STEP 4: Return payment link
    RETURN {
        payment_link: response.data.link,
        tx_ref: tx_ref
    }
```

### Verification Sequence

```
FUNCTION verify_payment(transaction_id, tx_ref):
    // STEP 1: Get order from database
    order = database.get_order_by_tx_ref(tx_ref)

    IF order IS NULL:
        THROW OrderNotFoundError(tx_ref)

    // STEP 2: Idempotency check
    IF order.status == "paid":
        RETURN { success: true, message: "Already paid" }

    // STEP 3: Verify with Flutterwave
    response = flutterwave.verify_transaction(transaction_id)

    IF response.status != "success":
        THROW VerificationError(response.message)

    transaction = response.data

    // STEP 4: Validate status
    IF transaction.status == "pending" OR transaction.status == "success-pending-validation":
        RETURN { success: false, message: "Payment pending", wait_for_webhook: true }

    IF transaction.status != "successful":
        database.update_order(order.id, { status: "failed" })
        RETURN { success: false, message: "Payment failed" }

    // STEP 5: Validate amount (CRITICAL)
    IF transaction.amount != order.amount:
        database.update_order(order.id, { status: "amount_mismatch" })
        THROW AmountMismatchError(order.amount, transaction.amount)

    // STEP 6: Validate currency
    IF transaction.currency != order.currency:
        database.update_order(order.id, { status: "currency_mismatch" })
        THROW CurrencyMismatchError(order.currency, transaction.currency)

    // STEP 7: Mark as paid
    database.update_order(order.id, {
        status: "paid",
        paid_at: now(),
        transaction_id: transaction.id,
        flw_ref: transaction.flw_ref
    })

    // STEP 8: Fulfill order
    fulfill_order(order)

    RETURN { success: true, transaction: transaction }
```

---

## Data Models

### Order Model

```
Order {
    id: UUID (primary key)
    tx_ref: STRING (unique, indexed)
    email: STRING
    amount: INTEGER (in smallest unit)
    currency: STRING (3 chars)
    status: ENUM ['pending', 'paid', 'failed', 'refunded', 'init_failed', 'amount_mismatch']
    transaction_id: INTEGER (nullable, from Flutterwave)
    flw_ref: STRING (nullable, from Flutterwave)
    metadata: JSON (nullable)
    created_at: DATETIME
    paid_at: DATETIME (nullable)
    updated_at: DATETIME
}

INDEXES:
    - tx_ref (unique)
    - email
    - status
    - created_at

```

### Webhook Event Model

```
WebhookEvent {
    id: UUID (primary key)
    event_type: STRING
    tx_ref: STRING (indexed)
    flw_ref: STRING (nullable)
    payload: JSON
    processed: BOOLEAN (default: false)
    processed_at: DATETIME (nullable)
    created_at: DATETIME
}

INDEXES:
    - tx_ref
    - event_type
    - processed
```

### Subscription Model (if using subscriptions)

```
Subscription {
    id: UUID (primary key)
    user_id: UUID (foreign key)
    plan_id: INTEGER
    flw_subscription_id: INTEGER
    status: ENUM ['active', 'cancelled', 'expired']
    current_period_start: DATETIME
    current_period_end: DATETIME
    created_at: DATETIME
    updated_at: DATETIME
}
```

---

## Service Layer

### Payment Service

```
CLASS PaymentService:
    CONSTRUCTOR(flutterwave_client, database):
        this.flw = flutterwave_client
        this.db = database

    METHOD initialize(email, amount, currency, metadata = null):
        tx_ref = generate_tx_ref()
        amount_in_smallest = to_smallest_unit(amount)

        // Create order first
        order = this.db.orders.create({
            tx_ref: tx_ref,
            email: email,
            amount: amount_in_smallest,
            currency: currency,
            status: "pending",
            metadata: metadata
        })

        TRY:
            response = this.flw.payments.initialize({
                tx_ref: tx_ref,
                amount: amount_in_smallest,
                currency: currency,
                redirect_url: CONFIG.callback_url,
                customer: { email: email },
                meta: metadata
            })

            RETURN { link: response.data.link, tx_ref: tx_ref }

        CATCH error:
            this.db.orders.update(order.id, { status: "init_failed" })
            THROW error

    METHOD verify(transaction_id_or_tx_ref):
        // Determine if ID or tx_ref
        IF is_numeric(transaction_id_or_tx_ref):
            transaction = this.flw.transactions.verify(transaction_id_or_tx_ref)
        ELSE:
            transaction = this.flw.transactions.verify_by_reference(transaction_id_or_tx_ref)

        order = this.db.orders.find_by_tx_ref(transaction.tx_ref)

        IF order IS NULL:
            THROW OrderNotFoundError()

        IF order.status == "paid":
            RETURN { already_paid: true, order: order }

        // Validate
        this.validate_transaction(transaction, order)

        // Update order
        this.db.orders.update(order.id, {
            status: "paid",
            paid_at: now(),
            transaction_id: transaction.id,
            flw_ref: transaction.flw_ref
        })

        RETURN { success: true, order: order, transaction: transaction }

    PRIVATE METHOD validate_transaction(transaction, order):
        IF transaction.status != "successful":
            THROW PaymentNotSuccessfulError(transaction.status)

        IF transaction.amount != order.amount:
            THROW AmountMismatchError(order.amount, transaction.amount)

        IF transaction.currency != order.currency:
            THROW CurrencyMismatchError(order.currency, transaction.currency)
```

### Webhook Service

```
CLASS WebhookService:
    CONSTRUCTOR(secret_hash, database, payment_service):
        this.secret_hash = secret_hash
        this.db = database
        this.payment_service = payment_service

    METHOD handle(headers, body):
        // Verify signature
        signature = headers["verif-hash"]
        IF signature != this.secret_hash:
            THROW InvalidSignatureError()

        event = parse_json(body)

        // Log event
        this.db.webhook_events.create({
            event_type: event.event,
            tx_ref: event.data.tx_ref,
            flw_ref: event.data.flw_ref,
            payload: event
        })

        // Route to handler
        SWITCH event.event:
            CASE "charge.completed":
                this.handle_charge_completed(event.data)
            CASE "charge.failed":
                this.handle_charge_failed(event.data)
            CASE "transfer.completed":
                this.handle_transfer_completed(event.data)
            CASE "transfer.failed":
                this.handle_transfer_failed(event.data)
            DEFAULT:
                log("Unhandled event: " + event.event)

    PRIVATE METHOD handle_charge_completed(data):
        tx_ref = data.tx_ref

        // Idempotency
        order = this.db.orders.find_by_tx_ref(tx_ref)
        IF order IS NULL OR order.status == "paid":
            RETURN

        IF data.status != "successful":
            RETURN

        // Validate and fulfill
        TRY:
            this.payment_service.verify(data.id)
            this.fulfill_order(order)
        CATCH error:
            log("Error fulfilling order: " + error)

    PRIVATE METHOD handle_charge_failed(data):
        tx_ref = data.tx_ref
        order = this.db.orders.find_by_tx_ref(tx_ref)

        IF order IS NOT NULL AND order.status == "pending":
            this.db.orders.update(order.id, {
                status: "failed",
                metadata: { failure_reason: data.processor_response }
            })
```

---

## Webhook Processing

### Idempotent Processing Pattern

```
FUNCTION process_webhook_idempotently(event):
    // Generate idempotency key
    idempotency_key = event.data.tx_ref + ":" + event.event

    // Check if already processed
    existing = database.processed_events.find(idempotency_key)
    IF existing IS NOT NULL:
        log("Event already processed: " + idempotency_key)
        RETURN { skipped: true }

    // Process event in transaction
    database.transaction(function(tx):
        // Double-check within transaction
        IF tx.processed_events.find(idempotency_key):
            RETURN

        // Process the event
        result = process_event(event)

        // Mark as processed
        tx.processed_events.create({
            key: idempotency_key,
            event_type: event.event,
            processed_at: now()
        })

        RETURN result
    )
```

### Async Processing Pattern

```
FUNCTION handle_webhook_async(request):
    // Verify signature first
    IF NOT verify_signature(request):
        RETURN 401

    // Queue for processing
    queue.enqueue("webhook_processor", {
        event: request.body,
        received_at: now()
    })

    // Return immediately
    RETURN 200

// Background worker
WORKER webhook_processor:
    FUNCTION process(job):
        event = job.data.event

        TRY:
            process_webhook_idempotently(event)
        CATCH error:
            // Log but don't retry for business logic errors
            IF error IS BusinessLogicError:
                log("Business error: " + error)
            ELSE:
                // Retry for transient errors
                THROW error
```

---

## Error Handling

### Error Types

```
// Base error
CLASS FlutterwaveError EXTENDS Error:
    code: STRING
    details: OBJECT

// Specific errors
CLASS PaymentInitError EXTENDS FlutterwaveError:
    code = "PAYMENT_INIT_FAILED"

CLASS VerificationError EXTENDS FlutterwaveError:
    code = "VERIFICATION_FAILED"

CLASS AmountMismatchError EXTENDS FlutterwaveError:
    code = "AMOUNT_MISMATCH"
    expected: INTEGER
    received: INTEGER

CLASS CurrencyMismatchError EXTENDS FlutterwaveError:
    code = "CURRENCY_MISMATCH"
    expected: STRING
    received: STRING

CLASS InvalidSignatureError EXTENDS FlutterwaveError:
    code = "INVALID_SIGNATURE"

CLASS OrderNotFoundError EXTENDS FlutterwaveError:
    code = "ORDER_NOT_FOUND"
```

### Error Handler Pattern

```
FUNCTION handle_payment_error(error, context):
    SWITCH error.code:
        CASE "PAYMENT_INIT_FAILED":
            // Log and notify
            log_error(error, context)
            notify_admin("Payment init failed", error)
            RETURN { status: 500, message: "Unable to process payment" }

        CASE "AMOUNT_MISMATCH":
            // This is potential fraud - log and alert
            log_security_event(error, context)
            alert_security_team(error)
            RETURN { status: 400, message: "Payment verification failed" }

        CASE "INVALID_SIGNATURE":
            // Log attempt
            log_security_event(error, context)
            RETURN { status: 401, message: "Invalid request" }

        CASE "ORDER_NOT_FOUND":
            RETURN { status: 404, message: "Order not found" }

        DEFAULT:
            log_error(error, context)
            RETURN { status: 500, message: "An error occurred" }
```

---

## State Machine

### Order State Machine

```
ORDER_STATES:
    pending
    paid
    failed
    refunded
    init_failed
    amount_mismatch
    expired

ORDER_TRANSITIONS:
    pending -> paid          (on successful verification)
    pending -> failed        (on failed charge)
    pending -> init_failed   (on API error during init)
    pending -> amount_mismatch (on amount validation failure)
    pending -> expired       (on timeout, e.g., 24 hours)
    paid -> refunded         (on refund request)

FUNCTION transition_order(order, new_status):
    allowed = get_allowed_transitions(order.status)

    IF new_status NOT IN allowed:
        THROW InvalidTransitionError(order.status, new_status)

    // Update with audit
    database.orders.update(order.id, {
        status: new_status,
        updated_at: now()
    })

    database.order_history.create({
        order_id: order.id,
        from_status: order.status,
        to_status: new_status,
        changed_at: now()
    })

    // Trigger side effects
    SWITCH new_status:
        CASE "paid":
            trigger_fulfillment(order)
            send_confirmation_email(order)
        CASE "failed":
            send_failure_notification(order)
        CASE "refunded":
            send_refund_notification(order)

FUNCTION get_allowed_transitions(current_status):
    SWITCH current_status:
        CASE "pending":
            RETURN ["paid", "failed", "init_failed", "amount_mismatch", "expired"]
        CASE "paid":
            RETURN ["refunded"]
        CASE "failed":
            RETURN []  // Terminal state
        CASE "refunded":
            RETURN []  // Terminal state
        DEFAULT:
            RETURN []
```

---

## Utility Functions

### Reference Generation

```
FUNCTION generate_tx_ref(prefix = "FLW"):
    timestamp = base36_encode(current_timestamp_millis())
    random = random_hex_string(8)
    RETURN uppercase(prefix + "_" + timestamp + "_" + random)

// Example output: FLW_LK5J2M8_A1B2C3D4
```

### Amount Conversion

```
FUNCTION to_smallest_unit(amount):
    RETURN round(amount * 100)

FUNCTION from_smallest_unit(amount):
    RETURN amount / 100

// Currency-specific (if needed)
FUNCTION to_smallest_unit_for_currency(amount, currency):
    multiplier = get_currency_multiplier(currency)  // All are 100 for supported currencies
    RETURN round(amount * multiplier)
```

### Signature Verification

```
// Simple verif-hash method
FUNCTION verify_signature_simple(signature, secret_hash):
    RETURN signature == secret_hash

// HMAC-SHA256 method
FUNCTION verify_signature_hmac(payload, signature, secret_hash):
    expected = base64_encode(
        hmac_sha256(secret_hash, payload)
    )

    // Timing-safe comparison
    RETURN constant_time_compare(expected, signature)
```
