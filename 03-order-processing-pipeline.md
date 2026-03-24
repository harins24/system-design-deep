# Order Processing Pipeline — Deep Dive Design

> **Scenario:** 10,000 orders/day through a multi-step pipeline:
> Place → Validate Inventory → Reserve (15 min) → Payment → Confirm/Cancel → Ship
> Notify customer · Update loyalty points · Notify warehouse · Support cancellation

**Tech Stack:** Java 17 · Spring Boot 3 · PostgreSQL · MongoDB · Redis · Kafka · Resilience4j

---

## Table of Contents

1. [Requirements & Constraints](#1-requirements--constraints)
2. [Capacity Estimation](#2-capacity-estimation)
3. [High-Level Architecture](#3-high-level-architecture)
4. [Core Design Questions Answered](#4-core-design-questions-answered)
   - 4.1 What microservices are involved?
   - 4.2 Synchronous or asynchronous communication?
   - 4.3 How do you ensure exactly-once payment processing?
   - 4.4 How do you implement inventory reservation with timeout?
   - 4.5 What happens if the email service is down?
   - 4.6 How do you handle payment gateway timeouts?
5. [Order State Machine](#5-order-state-machine)
6. [Saga Pattern — Orchestration vs Choreography](#6-saga-pattern--orchestration-vs-choreography)
7. [Outbox Pattern — Guaranteed Event Publishing](#7-outbox-pattern--guaranteed-event-publishing)
8. [Database Schema Design](#8-database-schema-design)
9. [Redis Data Structures](#9-redis-data-structures)
10. [Kafka Event Flow](#10-kafka-event-flow)
11. [Implementation Code](#11-implementation-code)
    - 11.1 Order Service — Saga Orchestrator
    - 11.2 Inventory Service — Reservation with TTL
    - 11.3 Payment Service — Exactly-Once with Timeout Handling
    - 11.4 Outbox Relay (Transactional Outbox)
    - 11.5 Notification Service — Resilient Fan-out
    - 11.6 Cancellation Service
    - 11.7 Reservation Expiry Worker
12. [Failure Scenarios & Mitigations](#12-failure-scenarios--mitigations)
13. [Scaling Strategy](#13-scaling-strategy)
14. [Monitoring & Observability](#14-monitoring--observability)
15. [Summary Cheat Sheet](#15-summary-cheat-sheet)

---

## 1. Requirements & Constraints

### Functional Requirements

**Happy path:**
```
Order Placed
  → Validate inventory (is the item in stock?)
  → Reserve inventory (hold it for 15 minutes)
  → Process payment (charge the customer)
  → Confirm order + permanently reduce inventory + create shipment
  → Send confirmation email & SMS
  → Update loyalty points
  → Notify warehouse for fulfillment
```

**Failure paths:**
```
Payment fails    → Release inventory reservation → Notify customer
Payment timeout  → Re-query gateway status → decide confirm or release
Order cancelled  → If pre-shipment: release reservation + issue refund
                 → If post-shipment: initiate return flow (out of scope)
Reservation expires (15 min) → Cancel unpaid order → Release inventory
```

### Non-Functional Requirements

- **Orders must not be lost** — durable from the moment of placement; survives any single node crash
- **No double-charging** — exactly-once payment semantics; retry safe
- **Inventory accuracy** — no overselling; reservations must be atomic
- **Payment gateway latency** — gateways can take up to 30 seconds; must not block other orders
- **Email resilience** — email service downtime must not block order confirmation
- **Order cancellation** — supported any time before shipment is dispatched
- **Audit trail** — every state transition logged immutably for compliance/support

### Out of Scope

- Checkout and cart (covered in Question 2)
- Returns / refunds post-delivery
- Multi-seller marketplace splits
- Tax calculation

---

## 2. Capacity Estimation

```
Orders:            10,000 orders/day
                   ~7 orders/second average
                   ~35 orders/second peak (5x)

Steps per order:   ~8 (validate, reserve, pay, confirm, ship, notify×3)
Total events:      10,000 × 8 = 80,000 Kafka events/day = ~1 event/second avg

Payment duration:  Avg 2s, p99 30s (gateway timeout)
Reservation hold:  15 minutes = 900 seconds

Inventory reservations active at any time:
  35 orders/sec × 900 sec hold = ~31,500 active reservations
  Each reservation ~200 bytes in Redis → 6.3 MB (trivial)

PostgreSQL write load:
  10,000 orders × ~5 DB writes each = 50,000 writes/day = ~0.6 writes/sec avg
  Peak: 2.5 writes/sec — very light

Kafka throughput:
  ~1 event/sec avg, 5/sec peak — trivially handled
  Kafka designed for millions/sec — this is a light workload

Storage:
  Orders (PostgreSQL):          10,000/day × 2 KB = 20 MB/day → 7 GB/year
  Order items (PostgreSQL):     50,000 items/day × 500 B = 25 MB/day
  Outbox events (PostgreSQL):   80,000/day × 1 KB = 80 MB/day (purge after relay)
  Order history (MongoDB):      10,000/day × 5 KB = 50 MB/day (audit trail)
  Redis reservations:           31,500 active × 200 B = ~6 MB
```

---

## 3. High-Level Architecture

```
                    ┌──────────────────────────────────────────────────────┐
                    │                  CLIENT LAYER                        │
                    │         Browser / Mobile / Partner API               │
                    └─────────────────────┬────────────────────────────────┘
                                          │ POST /orders
                    ┌─────────────────────▼────────────────────────────────┐
                    │            API GATEWAY                               │
                    │   JWT validation · Rate limiting · Routing           │
                    └─────────────────────┬────────────────────────────────┘
                                          │
                    ┌─────────────────────▼────────────────────────────────┐
                    │              ORDER SERVICE                           │
                    │     ★ Saga Orchestrator (owns the workflow)          │
                    │                                                      │
                    │  1. Validate + persist order (PENDING)               │
                    │  2. Write to Outbox (transactional)                  │
                    │  3. Listen to Kafka events for saga steps            │
                    │  4. Advance saga state or trigger compensation       │
                    └──────────────┬───────────────────────────────────────┘
                                   │ Kafka Commands + Events
          ┌────────────────────────┼────────────────────────────────┐
          │                        │                                │
┌─────────▼──────────┐  ┌──────────▼──────────┐  ┌────────────────▼────────┐
│ INVENTORY SERVICE  │  │  PAYMENT SERVICE    │  │  SHIPMENT SERVICE       │
│ - Reserve units    │  │  - Charge gateway   │  │  - Create shipment      │
│ - Release on fail  │  │  - Idempotency key  │  │  - Assign carrier       │
│ - Confirm on pay   │  │  - Timeout polling  │  │  - Track status         │
│ - Redis holds      │  │  - Stripe / Braintree│  │  - Dispatch event      │
└────────────────────┘  └─────────────────────┘  └─────────────────────────┘

          ┌──────────────────────────────────────────────────────────┐
          │                       KAFKA                              │
          │  order-commands · order-events · payment-events          │
          │  inventory-events · shipment-events · notification-events│
          └──────┬──────────────┬──────────────┬─────────────────────┘
                 │              │              │
   ┌─────────────▼──┐  ┌────────▼──────┐  ┌───▼──────────────────────┐
   │ NOTIFICATION   │  │  LOYALTY SVC  │  │  WAREHOUSE NOTIFICATION  │
   │ SERVICE        │  │  Update points│  │  SERVICE                 │
   │ Email + SMS    │  │               │  │  WMS integration         │
   │ Outbox pattern │  └───────────────┘  └──────────────────────────┘
   └────────────────┘

Data Stores:
  ┌──────────────────┐  ┌────────────────────┐  ┌──────────────────────┐
  │   POSTGRESQL     │  │      REDIS         │  │      MONGODB         │
  │  orders          │  │  reservations      │  │  order_history       │
  │  order_items     │  │  idempotency keys  │  │  (immutable audit)   │
  │  saga_state      │  │  payment locks     │  │  customer timeline   │
  │  outbox_events   │  │  reservation expiry│  │                      │
  │  inventory       │  │  saga locks        │  │                      │
  └──────────────────┘  └────────────────────┘  └──────────────────────┘
```

---

## 4. Core Design Questions Answered

---

### 4.1 What Microservices Are Involved?

**Decompose by business capability — each service owns one domain end-to-end.**

```
Service                  Responsibility                           DB
────────────────────────────────────────────────────────────────────────────
order-service            Saga orchestration, order lifecycle      PostgreSQL
                         Own: orders, order_items, saga_state

inventory-service        Stock management, reservations           PostgreSQL
                         Own: inventory, reservations             + Redis

payment-service          Gateway integration, charge/refund       PostgreSQL
                         Own: payments, payment_attempts

shipment-service         Shipment creation, tracking              PostgreSQL
                         Own: shipments, tracking_events

notification-service     Email + SMS delivery                     PostgreSQL
                         Own: notification_log, outbox

loyalty-service          Points earning + redemption              PostgreSQL
                         Own: loyalty_accounts, transactions

warehouse-service        WMS integration, fulfillment tasks       PostgreSQL
                         Own: fulfillment_tasks

order-query-service      Read-optimized order history             MongoDB
                         Own: order_history (denormalized)
```

**Key design principle:** Order-service is the **orchestrator** — it knows the workflow steps and drives them. Other services are **workers** — they execute one step when commanded and report back. This means:
- Only order-service knows the full saga workflow
- Other services are decoupled — they don't know about "orders," only their own domain ("charge this payment," "reserve this stock")
- Easy to add/remove steps without changing worker services

---

### 4.2 Synchronous or Asynchronous Communication?

**The rule: use synchronous (HTTP/gRPC) only where you need an immediate answer that blocks the user. Use asynchronous (Kafka) for everything else in the order pipeline.**

```
SYNCHRONOUS (blocking REST call):
  ✅ Order validation at placement time
     → User is waiting; need immediate feedback on invalid items
     → Example: product discontinued, address invalid → fail fast

  ✅ Inventory pre-check (soft check, not the reservation)
     → Quick stock availability check before accepting order
     → If out of stock: reject with 400 immediately
     → If in stock: accept order and start async pipeline

  ✅ Payment status query (polling for timeout resolution)
     → Described in 4.6; sometimes need to ask gateway "what happened?"

ASYNCHRONOUS (Kafka):
  ✅ Inventory reservation  → user already has order ID; can wait
  ✅ Payment processing     → 30s timeout; must not block HTTP thread
  ✅ Order confirmation     → downstream of payment
  ✅ Email / SMS            → fire-and-forget; failure is non-blocking
  ✅ Loyalty point update   → non-critical; acceptable delay
  ✅ Warehouse notification → warehouse processes in batches anyway

Why Kafka for most steps?
  Payment can take 30 seconds → HTTP thread would be held for 30s per order
  At 35 orders/sec peak = 35 × 30 = 1,050 threads blocked simultaneously
  Spring Boot default thread pool: 200 → system deadlocks
  Solution: Accept HTTP request immediately → return 202 Accepted
           Process payment asynchronously via Kafka event loop
```

**Communication decision matrix:**

| Step | Pattern | Why |
|---|---|---|
| POST /orders | REST sync | User waits; validate + persist + return 202 |
| Inventory reserve | Kafka async | Command → InventoryService; no blocking needed |
| Payment charge | Kafka async | 30s gateway; must not block HTTP thread |
| Payment timeout re-query | REST sync | Need immediate answer from gateway to decide |
| Confirm/cancel | Kafka async | Downstream of payment result |
| Email/SMS | Kafka async | Fire-and-forget; non-critical path |
| Loyalty update | Kafka async | Eventually consistent; acceptable lag |
| Warehouse notify | Kafka async | Batch processed by WMS |

---

### 4.3 How Do You Ensure Exactly-Once Payment Processing?

**The golden rule: a customer must never be charged twice for the same order, even if the network drops, the service crashes, or the user hits "pay" twice.**

**Three layers of protection:**

**Layer 1: Idempotency Key (client → payment-service)**

```
Every payment request carries a unique idempotency key:
  Key = "payment:" + orderId                  (order-scoped)
  Or:  "payment:" + orderId + ":" + attemptNo (attempt-scoped)

First call:
  payment-service receives key → checks Redis/DB → not seen before
  → Process the charge → store result under key → return result

Retry call (same key):
  payment-service receives key → checks Redis/DB → found!
  → Return stored result → NEVER re-charge the gateway

Key storage:
  Redis: "idempotency:{key}" → response JSON, TTL 24h
         Fast lookup, acceptable to lose after 24h (client should retry within hours)
  PostgreSQL: payments table has UNIQUE constraint on idempotency_key
              Permanent record; catches Redis eviction edge case
```

**Layer 2: Outbox Pattern (payment-service → Kafka)**

```
Problem: payment-service charges card successfully → crashes before publishing Kafka event
         Order-service never gets PaymentSucceeded event → order stuck in PAYMENT_PENDING

Solution: Transactional Outbox
  payment-service's local PostgreSQL transaction:
    BEGIN TRANSACTION;
      INSERT INTO payments (order_id, amount, status, gateway_txn_id) ...  ← charge result
      INSERT INTO outbox_events (topic, payload) ...                        ← Kafka event
    COMMIT TRANSACTION;                                                     ← atomic!

  If crash BEFORE commit: both records rolled back → no charge, no event ✅
  If crash AFTER commit:  both records persisted → outbox relay will publish event ✅
  The charge result and the Kafka event are ALWAYS in sync
```

**Layer 3: Gateway Idempotency (payment-service → Stripe/Braintree)**

```
Major payment gateways (Stripe, Braintree, Adyen) support idempotency keys natively:
  Stripe: pass idempotency_key header
  If gateway receives duplicate call with same key within 24h:
    → Returns original response, does NOT charge again

This handles:
  payment-service retrying after a network error
  payment-service restarting mid-request
  Duplicate Kafka message delivery causing re-processing

Complete protection chain:
  Client → [idempotency key] → payment-service → [gateway idempotency key] → Stripe
  Result ← [cached response]  ←                  [cached gateway response] ←
```

**Idempotency implementation detail:**

```java
// Before charging: check for existing result
public PaymentResult processPayment(String orderId, long amountCents) {
    String idemKey = "payment:" + orderId;

    // Check Redis first (fast path)
    String cached = redis.opsForValue().get("idempotency:" + idemKey);
    if (cached != null) return deserialize(cached, PaymentResult.class);

    // Check DB (slow path — catches Redis miss/eviction)
    Optional<Payment> existing = paymentRepo.findByIdempotencyKey(idemKey);
    if (existing.isPresent()) return toResult(existing.get());

    // Lock to prevent concurrent duplicates for same orderId
    String lockKey = "payment:lock:" + orderId;
    Boolean acquired = redis.opsForValue().setIfAbsent(lockKey, "1", Duration.ofSeconds(60));
    if (Boolean.FALSE.equals(acquired)) {
        throw new PaymentInProgressException("Payment already in progress for order " + orderId);
    }

    try {
        // Charge the gateway
        GatewayResponse response = gateway.charge(ChargeRequest.builder()
            .amount(amountCents)
            .idempotencyKey(idemKey)  // gateway-level dedup
            .build());

        // Persist (transactional outbox — atomic with Kafka event)
        PaymentResult result = saveWithOutbox(orderId, amountCents, idemKey, response);

        // Cache in Redis for fast idempotency checks
        redis.opsForValue().set("idempotency:" + idemKey,
            serialize(result), Duration.ofHours(24));

        return result;
    } finally {
        redis.delete(lockKey);
    }
}
```

---

### 4.4 How Do You Implement Inventory Reservation with Timeout?

**Two-phase inventory management: Reserve (soft hold) → Confirm (permanent deduct) or Release (undo hold).**

```
Phase 1 — Reserve (at order placement):
  Inventory decremented by "reserved" amount (not sold yet)
  Physical stock = available_qty + reserved_qty
  Available to other buyers = available_qty - reserved_qty

  available_qty: 10
  Order A reserves 2 → available_qty: 8, reserved_qty: 2
  Order B reserves 3 → available_qty: 5, reserved_qty: 5
  Order C tries 6   → available_qty only 5 → REJECTED ✅ (no oversell)

Phase 2A — Confirm (payment succeeded):
  reserved_qty decremented permanently
  physical_qty decremented (item physically leaving warehouse)
  reserved_qty: 5 → 3 (released Order A's 2-unit hold)
  physical_qty:  10 → 8 (sold 2 units to Order A)

Phase 2B — Release (payment failed OR 15-min timeout):
  reserved_qty decremented (hold released, item available again)
  available_qty increases back
  reserved_qty: 5 → 3 (released Order A's 2-unit hold)
  available_qty: 5 → 7 (2 units back on shelf)
```

**Reservation storage — Redis + PostgreSQL:**

```
Redis (fast, TTL-driven):
  Key: "reservation:{reservationId}"
  Type: Hash
  Fields:
    orderId, skuId, quantity, reservedAt, expiresAt, status
  TTL: 15 minutes (900 seconds)
  When TTL fires: keyspace event → expiry worker releases the hold

  Key: "inventory:{skuId}:available"
  Value: integer (available quantity)
  Operations: DECRBY (reserve), INCRBY (release), all atomic

PostgreSQL (durable):
  reservations table: permanent record of all holds
  inventory table: authoritative stock levels
  Both updated in the same transaction (consistency guaranteed)
```

**Atomic reservation with Lua script:**

```lua
-- Atomically: check available stock, then reserve if sufficient
-- KEYS[1] = inventory available key  "inventory:{skuId}:available"
-- KEYS[2] = reservation key          "reservation:{reservationId}"
-- ARGV[1] = quantity to reserve
-- ARGV[2] = reservation TTL (seconds)
-- ARGV[3] = reservation data JSON
-- Returns: {1, remaining} on success, {-1, available} if insufficient

local available = tonumber(redis.call('GET', KEYS[1]) or '0')
local qty = tonumber(ARGV[1])

if available < qty then
    return {-1, available}  -- insufficient stock
end

-- Decrement available
local remaining = redis.call('DECRBY', KEYS[1], qty)

-- Store reservation with TTL
redis.call('SET', KEYS[2], ARGV[3], 'EX', ARGV[2])

return {1, remaining}
```

**15-minute timeout — three mechanisms working together:**

```
Mechanism 1: Redis TTL (primary, automatic)
  reservation key has TTL = 900 seconds
  When Redis expires the key:
    → Keyspace expiry event fires
    → Expiry worker: INCRBY inventory:{skuId}:available qty
    → Update PostgreSQL reservation status = EXPIRED
    → Publish ReservationExpired Kafka event
    → Order saga moves to CANCELLED state

Mechanism 2: Scheduled polling (safety net)
  Every 60 seconds, DB query:
    SELECT * FROM reservations
    WHERE status = 'ACTIVE'
      AND expires_at < NOW()
    → Expire each found reservation
  Catches cases where Redis keyspace events were missed

Mechanism 3: Order saga timeout check (defense in depth)
  When order-service processes payment result:
    If PaymentSucceeded event received BUT reservation is EXPIRED:
      → Issue refund immediately
      → Order moves to REFUND_PENDING (not CONFIRMED)
  Prevents confirming an order whose inventory was already released
```

---

### 4.5 What Happens If the Email Service Is Down?

**Email is non-blocking — its failure must never affect order confirmation.**

**Design: Outbox Pattern + Kafka + Retry with Dead Letter Queue (DLQ)**

```
Order-service flow:
  PaymentSucceeded received
  → Confirm order in DB (CONFIRMED)
  → Publish OrderConfirmed Kafka event
  → DONE — order-service's job is finished
  → Does NOT call email-service directly

Notification-service (Kafka consumer):
  Consumes OrderConfirmed event
  → Sends email via SendGrid / SES
  → If email service is DOWN:
     ✅ Kafka offset NOT committed → message stays in Kafka
     ✅ Kafka retries delivery automatically
     ✅ Customer email eventually sent when service recovers
     ✅ Order status unaffected — order is CONFIRMED regardless

Retry strategy for notifications:
  Attempt 1: immediate
  Attempt 2: 30 seconds later
  Attempt 3: 5 minutes later
  Attempt 4: 30 minutes later
  Attempt 5: 2 hours later
  After 5 failures: move to Dead Letter Queue (DLQ)

Dead Letter Queue:
  Topic: notification-dlq
  Operations team monitors DLQ
  Manual replay or special handling
  Typical causes: invalid email address, content blocked by spam filter

Notification deduplication:
  notification_log table: UNIQUE (order_id, notification_type, channel)
  If email is eventually sent twice (Kafka at-least-once): second insert violates constraint
  → Caught, logged, second send skipped ✅
```

**Notification service own outbox:**

```
notification-service cannot just call SendGrid and hope for the best.
It uses its own Outbox pattern too:

  notification-service receives OrderConfirmed Kafka event
  → Writes to its own outbox:
    BEGIN TRANSACTION;
      INSERT INTO notification_log (order_id, type, status = 'PENDING') ...
      INSERT INTO outbox_events (payload: {to, subject, body}) ...
    COMMIT;
  → Outbox relay reads outbox_events → calls SendGrid API
  → On SendGrid 200: UPDATE notification_log status = 'SENT'
  → On SendGrid 5xx: retry (exponential backoff)
  → On SendGrid 4xx (invalid email): mark FAILED, publish to DLQ

This ensures: if notification-service crashes mid-processing,
the notification will be retried from PostgreSQL state, not re-processed
from scratch based on Kafka offset alone.
```

---

### 4.6 How Do You Handle Payment Gateway Timeouts?

**Payment gateways can take up to 30 seconds. Never treat a timeout as a failure — the charge may have gone through!**

```
The danger of treating timeout = failure:
  payment-service calls Stripe → waits 30s → timeout exception
  payment-service publishes PaymentFailed event
  inventory released, order cancelled
  Customer receives "payment failed" email

  Meanwhile: Stripe DID charge the card (network response was just lost)
  Customer charged, no order → DISASTER (chargeback guaranteed)

Correct timeout handling:
  Timeout is UNKNOWN, not FAILED
  "I don't know if the charge went through" ≠ "the charge failed"
```

**The two-phase payment resolution protocol:**

```
Phase 1 — Attempt:
  payment-service sends charge request to gateway
  Sets hard timeout: 35 seconds (5s margin over 30s gateway limit)
  Case A: Response received < 35s → success or failure → proceed normally
  Case B: No response after 35s → enter UNKNOWN state

Phase 2 — Resolution (for UNKNOWN state):
  payment-service stores: payment status = UNKNOWN
  Does NOT publish any payment event yet

  After 5 seconds: poll gateway status API:
    GET /v1/charges/{chargeId}/status (synchronous, fast, <500ms)
    → SUCCESS: publish PaymentSucceeded → proceed with order
    → FAILED:  publish PaymentFailed  → release inventory
    → PENDING: still processing → poll again in 10s (max 5 polls)

  After 5 polls (55 seconds total): still UNKNOWN
    → Mark payment as TIMEOUT_UNRESOLVED
    → Release inventory reservation (safety first)
    → Put payment in manual review queue
    → Notify customer: "Your order is being reviewed"
    → Operations team resolves manually within 1 hour
```

**Implementation with scheduled resolution job:**

```java
// payment-service: attempt + async resolution
@Transactional
public void initiatePayment(InitiatePaymentCommand cmd) {
    String idemKey = "payment:" + cmd.getOrderId();

    // Save PENDING payment record + outbox event atomically
    Payment payment = Payment.builder()
        .orderId(cmd.getOrderId())
        .amountCents(cmd.getAmountCents())
        .status(PaymentStatus.PENDING)
        .idempotencyKey(idemKey)
        .build();
    paymentRepo.save(payment);

    // Publish to outbox → triggers async charge attempt
    outboxRepo.save(OutboxEvent.of("payment-commands", PaymentChargeCommand.of(cmd)));
}

// Async charge execution (runs in separate thread pool — not blocking Kafka consumer)
@Async("paymentExecutor")
public void executeCharge(String paymentId) {
    Payment payment = paymentRepo.findById(paymentId).orElseThrow();

    try {
        // Attempt with 35s timeout
        GatewayResponse response = gateway.charge(
            buildChargeRequest(payment),
            Duration.ofSeconds(35)
        );
        handleGatewayResponse(payment, response);

    } catch (GatewayTimeoutException e) {
        log.warn("Gateway timeout for payment {}, entering resolution mode", paymentId);
        // Store charge attempt ID from gateway (sent before timeout)
        payment.setGatewayAttemptId(e.getAttemptId());
        payment.setStatus(PaymentStatus.UNKNOWN);
        payment.setResolutionAttempts(0);
        payment.setNextResolutionAt(Instant.now().plusSeconds(5));
        paymentRepo.save(payment);
        // Do NOT publish any event — resolution job will handle this
    }
}

// Scheduled resolution for UNKNOWN payments
@Scheduled(fixedDelay = 5000)  // every 5 seconds
public void resolveUnknownPayments() {
    List<Payment> unknowns = paymentRepo.findByStatusAndNextResolutionAtBefore(
        PaymentStatus.UNKNOWN, Instant.now());

    for (Payment payment : unknowns) {
        try {
            GatewayStatusResponse status =
                gateway.queryStatus(payment.getGatewayAttemptId());

            switch (status.getStatus()) {
                case SUCCEEDED -> resolveAsSuccess(payment);
                case FAILED    -> resolveAsFailure(payment);
                case PENDING   -> scheduleNextPoll(payment);
            }
        } catch (Exception e) {
            log.error("Resolution poll failed for payment {}", payment.getId(), e);
            scheduleNextPoll(payment);
        }
    }
}

private void scheduleNextPoll(Payment payment) {
    int attempts = payment.getResolutionAttempts() + 1;
    payment.setResolutionAttempts(attempts);

    if (attempts >= 5) {
        payment.setStatus(PaymentStatus.TIMEOUT_UNRESOLVED);
        payment.setNextResolutionAt(null);
        notifyOperationsTeam(payment);
        releaseInventoryReservation(payment.getOrderId());  // safety first
    } else {
        long[] delays = {5, 10, 15, 30, 60};  // seconds between polls
        payment.setNextResolutionAt(
            Instant.now().plusSeconds(delays[attempts - 1]));
    }
    paymentRepo.save(payment);
}
```

---

## 5. Order State Machine

```
                         POST /orders
                              │
                    ┌─────────▼──────────┐
                    │    VALIDATING      │  ← Sync: check address, items, user
                    └─────────┬──────────┘
                              │ Validation OK
                    ┌─────────▼──────────┐
                    │     PENDING        │  ← Order created, returned to client (202)
                    └─────────┬──────────┘
                              │ Kafka: ReserveInventoryCommand
                    ┌─────────▼──────────┐
                    │  INVENTORY_PENDING │
                    └─────────┬──────────┘
               ┌──────────────┼──────────────────┐
               │              │                   │
    Inventory  │     Timeout  │         No stock  │
    reserved   │     (15 min) │                   │
               │              │                   │
   ┌───────────▼──┐  ┌─────────▼──────┐  ┌────────▼──────┐
   │  PAYMENT_    │  │   CANCELLED    │  │   REJECTED    │
   │  PENDING     │  │  (timeout)     │  │  (out of stock│
   └───────────┬──┘  └────────────────┘  └───────────────┘
               │
               │ Kafka: PaymentChargeCommand
               │
    ┌──────────▼────────────────────────┐
    │         PAYMENT_PROCESSING        │  ← Can take up to 30s
    └──────────┬──────┬─────────────────┘
               │      │
    Success    │      │   Failed / UNKNOWN → resolve
               │      │
   ┌───────────▼──┐  ┌▼───────────────────────────┐
   │  CONFIRMING  │  │   PAYMENT_FAILED            │
   └───────────┬──┘  │   → Release inventory       │
               │      │   → Publish PaymentFailed  │
               │      └────────────────────────────┘
               │
       ┌───────┴──────────────────────────┐
       │ Atomic local transaction:         │
       │  - Order status → CONFIRMED       │
       │  - Inventory: reserve → confirmed │
       │  - Create Shipment record         │
       │  - Outbox: OrderConfirmed event   │
       └───────────────┬──────────────────┘
                       │
               ┌───────▼──────┐
               │  CONFIRMED   │◄──── Cancellation window OPEN
               └───────┬──────┘      (until shipment dispatched)
                       │
             ┌─────────▼──────────┐
             │    FULFILLING      │  ← Warehouse processing
             └─────────┬──────────┘
                       │
          ┌────────────┼──────────────┐
          │            │              │
  Cancel  │   Ship     │    Error     │
  request │ dispatched │              │
          │            │              │
   ┌──────▼────┐  ┌────▼───────┐  ┌──▼────────┐
   │CANCELLATION│  │  SHIPPED  │  │  FAILED   │
   │_REQUESTED │  └────────────┘  └───────────┘
   └──────┬────┘
          │ Refund issued
   ┌──────▼────┐
   │ CANCELLED │
   └───────────┘

State transition rules:
  Can cancel: PENDING, INVENTORY_PENDING, PAYMENT_PENDING, CONFIRMED, FULFILLING
  Cannot cancel: SHIPPED, CANCELLED, REJECTED (terminal states)
  CONFIRMED → CANCELLATION_REQUESTED needs refund workflow
```

---

## 6. Saga Pattern — Orchestration vs Choreography

**Choice: Orchestration for the main order pipeline. Choreography for downstream notifications.**

```
WHY ORCHESTRATION for the order pipeline:
  Complex multi-step workflow (8+ steps)
  Each step has conditional branching (e.g., payment can succeed/fail/timeout)
  Compensation logic is non-trivial (3 different compensation paths)
  Need a single place to see "what step is this order on?"
  Debugging: "why is this order stuck?" → check saga_state table

WHY NOT CHOREOGRAPHY for the order pipeline:
  Without a coordinator: hard to know which step we're on
  Compensation events can create cyclic chains
  Hard to add a new step (e.g., fraud check) without touching many services
  "Saga spaghetti" in large teams

WHY CHOREOGRAPHY for notifications/loyalty/warehouse:
  These are fire-and-forget — no compensation needed
  OrderConfirmed → notification, loyalty, warehouse react independently
  Each service doesn't care what the others do
  Adding a new downstream consumer = just create a new Kafka consumer group
```

**Saga orchestration — step-by-step execution:**

```
Step 1: order-service starts saga
  → Send ReserveInventoryCommand to inventory-service via Kafka

Step 2: inventory-service processes command
  → Reserve stock atomically (Redis + PostgreSQL)
  → Publish InventoryReservedEvent (success) or InventoryUnavailableEvent (failure)

Step 3: order-service receives event
  InventoryReservedEvent → update saga state → send PaymentChargeCommand
  InventoryUnavailableEvent → update saga state → end saga (REJECTED)

Step 4: payment-service processes command
  → Charge gateway (idempotent)
  → Publish PaymentSucceededEvent or PaymentFailedEvent

Step 5: order-service receives event
  PaymentSucceededEvent → update saga state → send ConfirmOrderCommand
  PaymentFailedEvent → update saga state → send ReleaseInventoryCommand (compensate)

Step 6a (success path): order-service processes ConfirmOrderCommand locally
  → Confirm order, create shipment, publish OrderConfirmedEvent

Step 6b (failure path): inventory-service receives ReleaseInventoryCommand
  → Release reservation → publish InventoryReleasedEvent
  → order-service: final saga state = CANCELLED

Step 7 (after OrderConfirmedEvent, choreography takes over):
  → notification-service: send email + SMS (independently)
  → loyalty-service: credit points (independently)
  → warehouse-service: create fulfillment task (independently)
```

**Saga state in PostgreSQL:**

```sql
saga_state table:
  saga_id          UUID        = orderId (1:1 relationship)
  current_step     VARCHAR     = INVENTORY_RESERVING | PAYMENT_PENDING | ...
  compensation_step VARCHAR    = null or step being compensated
  steps_log        JSONB       = [{step, status, timestamp}]
  started_at       TIMESTAMPTZ
  completed_at     TIMESTAMPTZ
  status           VARCHAR     = IN_PROGRESS | COMPLETED | COMPENSATING | FAILED
```

---

## 7. Outbox Pattern — Guaranteed Event Publishing

**The transactional outbox solves the "dual write" problem: how to atomically update the DB AND publish a Kafka event.**

```
The dual write problem:
  payment-service charges card → ✅
  payment-service publishes Kafka event → ❌ (crash before publish)
  Result: order stuck forever in PAYMENT_PROCESSING, card was charged

  OR:
  payment-service publishes Kafka event → ✅
  payment-service writes to DB → ❌ (crash before DB write)
  Result: Kafka says payment succeeded, DB has no record → ghost payment

Solution: Transactional Outbox

  Within ONE local PostgreSQL transaction:
    INSERT INTO payments (...) status = SUCCESS     ← business state
    INSERT INTO outbox_events (topic, payload, ...) ← event to publish

  If crash BEFORE commit: both roll back, nothing published
  If crash AFTER commit:  both committed, relay will publish
  The two records are ALWAYS atomic with each other

Outbox relay (separate process, runs every 100ms):
  SELECT * FROM outbox_events WHERE published = false ORDER BY id LIMIT 100
  → For each row: publish to Kafka
  → On Kafka ACK: UPDATE outbox_events SET published = true, published_at = NOW()
  → Uses Kafka transactions for exactly-once relay
```

**Outbox relay implementation:**

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class OutboxRelay {

    private final OutboxEventRepository outboxRepo;
    private final KafkaTemplate<String, Object> kafka;

    @Scheduled(fixedDelay = 100)  // poll every 100ms
    @Transactional
    public void relayPendingEvents() {
        List<OutboxEvent> pending = outboxRepo.findTop100ByPublishedFalseOrderByIdAsc();
        if (pending.isEmpty()) return;

        for (OutboxEvent event : pending) {
            try {
                // Send to Kafka synchronously (wait for broker ACK)
                kafka.send(event.getTopic(), event.getPartitionKey(), event.getPayload())
                    .get(5, TimeUnit.SECONDS);  // block max 5s for ACK

                event.setPublished(true);
                event.setPublishedAt(Instant.now());
                outboxRepo.save(event);

            } catch (Exception e) {
                log.error("Failed to relay outbox event {}: {}", event.getId(), e.getMessage());
                event.setRetryCount(event.getRetryCount() + 1);
                event.setLastError(e.getMessage());
                if (event.getRetryCount() > 10) {
                    event.setDeadLettered(true);  // give up, manual intervention
                }
                outboxRepo.save(event);
            }
        }
    }
}
```

**Why not Kafka Transactions directly?**

```
Kafka transactions (exactly-once producer) work within Kafka:
  Producer → Kafka: guaranteed exactly-once delivery to Kafka

But they don't span Kafka and PostgreSQL:
  "Charge the DB AND publish to Kafka atomically" → not possible with Kafka transactions

Outbox solves this:
  PostgreSQL transaction → atomic write of {business record + outbox row}
  Outbox relay → reads from PostgreSQL → publishes to Kafka
  Two separate operations, but database is the source of truth for "did we publish?"
```

---

## 8. Database Schema Design

### PostgreSQL — Transactional Core

```sql
-- Orders table
CREATE TABLE orders (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             UUID NOT NULL,
    status              VARCHAR(30) NOT NULL DEFAULT 'PENDING',
                        -- PENDING | INVENTORY_PENDING | PAYMENT_PENDING |
                        -- PAYMENT_PROCESSING | CONFIRMED | FULFILLING |
                        -- SHIPPED | CANCELLED | REJECTED | FAILED
    subtotal_cents      BIGINT NOT NULL,
    shipping_cents      BIGINT NOT NULL DEFAULT 0,
    tax_cents           BIGINT NOT NULL DEFAULT 0,
    total_cents         BIGINT NOT NULL,
    shipping_address_id UUID NOT NULL,
    store_id            UUID,
    idempotency_key     VARCHAR(128) UNIQUE NOT NULL,  -- prevent duplicate order placement
    version             INT NOT NULL DEFAULT 0,         -- optimistic locking
    created_at          TIMESTAMPTZ DEFAULT NOW(),
    confirmed_at        TIMESTAMPTZ,
    cancelled_at        TIMESTAMPTZ,
    cancellation_reason TEXT
);

CREATE INDEX idx_orders_user_id    ON orders(user_id, created_at DESC);
CREATE INDEX idx_orders_status     ON orders(status, created_at DESC);
CREATE INDEX idx_orders_idem_key   ON orders(idempotency_key);

-- Order items
CREATE TABLE order_items (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id        UUID NOT NULL REFERENCES orders(id),
    sku_id          VARCHAR(100) NOT NULL,
    product_id      UUID NOT NULL,
    product_name    VARCHAR(500) NOT NULL,   -- denormalized snapshot at order time
    quantity        INT NOT NULL,
    unit_price_cents BIGINT NOT NULL,        -- price locked at order time
    total_cents     BIGINT NOT NULL
);

CREATE INDEX idx_order_items_order ON order_items(order_id);

-- Saga state table
CREATE TABLE saga_state (
    saga_id         UUID PRIMARY KEY,        -- same as order_id (1:1)
    current_step    VARCHAR(50) NOT NULL,
    status          VARCHAR(30) NOT NULL,
                    -- IN_PROGRESS | COMPLETED | COMPENSATING | FAILED
    steps_log       JSONB NOT NULL DEFAULT '[]',
    started_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW(),
    completed_at    TIMESTAMPTZ
);

-- Inventory table
CREATE TABLE inventory (
    sku_id          VARCHAR(100) NOT NULL,
    store_id        UUID,                    -- NULL = central warehouse
    available_qty   INT NOT NULL DEFAULT 0,
    reserved_qty    INT NOT NULL DEFAULT 0,
    physical_qty    INT NOT NULL,            -- available + reserved
    updated_at      TIMESTAMPTZ DEFAULT NOW(),
    version         INT NOT NULL DEFAULT 0,  -- optimistic lock

    PRIMARY KEY (sku_id, COALESCE(store_id, '00000000-0000-0000-0000-000000000000'::UUID)),
    CONSTRAINT non_negative_qty CHECK (available_qty >= 0 AND reserved_qty >= 0)
);

-- Reservations table
CREATE TABLE inventory_reservations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id        UUID NOT NULL REFERENCES orders(id),
    sku_id          VARCHAR(100) NOT NULL,
    store_id        UUID,
    quantity        INT NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
                    -- ACTIVE | CONFIRMED | RELEASED | EXPIRED
    reserved_at     TIMESTAMPTZ DEFAULT NOW(),
    expires_at      TIMESTAMPTZ NOT NULL,    -- reserved_at + 15 minutes
    released_at     TIMESTAMPTZ

);

CREATE INDEX idx_reservations_order_id   ON inventory_reservations(order_id);
CREATE INDEX idx_reservations_status_exp ON inventory_reservations(status, expires_at)
    WHERE status = 'ACTIVE';  -- partial index for expiry job

-- Payments table
CREATE TABLE payments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id        UUID NOT NULL REFERENCES orders(id),
    idempotency_key VARCHAR(128) UNIQUE NOT NULL,
    gateway         VARCHAR(20) NOT NULL,    -- STRIPE | BRAINTREE | PAYPAL
    gateway_txn_id  VARCHAR(100),            -- gateway's transaction ID
    gateway_attempt_id VARCHAR(100),         -- for timeout resolution
    amount_cents    BIGINT NOT NULL,
    status          VARCHAR(30) NOT NULL,
                    -- PENDING | PROCESSING | SUCCEEDED | FAILED
                    -- UNKNOWN | TIMEOUT_UNRESOLVED | REFUNDED
    failure_code    VARCHAR(50),
    failure_message TEXT,
    resolution_attempts INT DEFAULT 0,
    next_resolution_at  TIMESTAMPTZ,
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    completed_at    TIMESTAMPTZ
);

CREATE INDEX idx_payments_order_id      ON payments(order_id);
CREATE INDEX idx_payments_status        ON payments(status, next_resolution_at)
    WHERE status = 'UNKNOWN';  -- partial index for resolution job

-- Shipments table
CREATE TABLE shipments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id        UUID NOT NULL REFERENCES orders(id),
    status          VARCHAR(30) NOT NULL DEFAULT 'PENDING',
                    -- PENDING | PICKING | PACKED | DISPATCHED | IN_TRANSIT | DELIVERED
    warehouse_id    UUID,
    carrier         VARCHAR(50),
    tracking_number VARCHAR(100),
    estimated_delivery DATE,
    dispatched_at   TIMESTAMPTZ,
    delivered_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- Transactional outbox
CREATE TABLE outbox_events (
    id              BIGSERIAL PRIMARY KEY,
    aggregate_id    UUID NOT NULL,           -- orderId, paymentId, etc.
    aggregate_type  VARCHAR(50) NOT NULL,    -- ORDER, PAYMENT, INVENTORY
    topic           VARCHAR(100) NOT NULL,
    partition_key   VARCHAR(200),
    event_type      VARCHAR(100) NOT NULL,
    payload         JSONB NOT NULL,
    published       BOOLEAN NOT NULL DEFAULT FALSE,
    published_at    TIMESTAMPTZ,
    retry_count     INT NOT NULL DEFAULT 0,
    last_error      TEXT,
    dead_lettered   BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_outbox_unpublished ON outbox_events(id)
    WHERE published = FALSE AND dead_lettered = FALSE;  -- partial index for relay query
```

### MongoDB — Audit Trail & Order History

```javascript
// Collection: order_history  (immutable, append-only per order)
{
  _id: ObjectId,
  orderId: "uuid",
  userId: "uuid",
  version: 8,   // latest version number
  status: "CONFIRMED",

  // Full order snapshot at current state
  items: [
    {
      skuId: "APPLE-IPHONE-15", productName: "iPhone 15 Pro",
      quantity: 1, unitPriceCents: 99900
    }
  ],
  totalCents: 99900,

  // Complete history of state transitions
  stateHistory: [
    { status: "PENDING",            occurredAt: "2026-03-20T10:00:00Z", actor: "USER" },
    { status: "INVENTORY_PENDING",  occurredAt: "2026-03-20T10:00:01Z", actor: "SYSTEM" },
    { status: "PAYMENT_PENDING",    occurredAt: "2026-03-20T10:00:02Z", actor: "SYSTEM" },
    { status: "PAYMENT_PROCESSING", occurredAt: "2026-03-20T10:00:03Z", actor: "SYSTEM" },
    { status: "CONFIRMED",          occurredAt: "2026-03-20T10:00:05Z", actor: "SYSTEM" }
  ],

  // Payment details snapshot
  payment: {
    paymentId: "uuid", gateway: "STRIPE",
    gatewayTxnId: "pi_xyz", amountCents: 99900,
    paidAt: "2026-03-20T10:00:05Z"
  },

  // Shipping details
  shipment: {
    shipmentId: "uuid", warehouseId: "uuid",
    carrier: "UPS", trackingNumber: "1Z999AA1234567890",
    estimatedDelivery: "2026-03-25"
  },

  updatedAt: ISODate("2026-03-20T10:00:05Z")
}

db.order_history.createIndex({ "orderId": 1 }, { unique: true })
db.order_history.createIndex({ "userId": 1, "updatedAt": -1 })
db.order_history.createIndex({ "status": 1, "updatedAt": -1 })
```

---

## 9. Redis Data Structures

```
Key Pattern                                    Type     TTL         Purpose
─────────────────────────────────────────────────────────────────────────────────
inventory:{skuId}:available                    STRING   None        Available quantity (atomic DECRBY)
inventory:{skuId}:{storeId}:available          STRING   None        Store-specific available qty
reservation:{reservationId}                    HASH     900s (15m)  Active reservation data
reservation:order:{orderId}                    STRING   900s        OrderId → reservationId mapping

payment:lock:{orderId}                         STRING   60s         Prevent concurrent payment attempts
payment:idempotency:{idemKey}                  STRING   24h         Cached payment result
payment:unknown:{paymentId}                    STRING   1h          UNKNOWN payment tracking

saga:lock:{orderId}                            STRING   30s         Prevent concurrent saga steps
saga:version:{orderId}                         STRING   24h         Saga optimistic lock version

idempotency:order:{idemKey}                    STRING   24h         Order placement dedup
cancellation:lock:{orderId}                    STRING   10s         Prevent concurrent cancellations
```

---

## 10. Kafka Event Flow

```
Topic                       Partitions  Key        Retention  Description
────────────────────────────────────────────────────────────────────────────────
order-commands              20          orderId    1 day      Commands FROM orchestrator
                                                              ReserveInventoryCommand
                                                              PaymentChargeCommand
                                                              ReleaseInventoryCommand
                                                              ConfirmOrderCommand

order-events                20          orderId    7 days     State events FROM order-service
                                                              OrderCreatedEvent
                                                              OrderConfirmedEvent
                                                              OrderCancelledEvent
                                                              OrderFailedEvent

inventory-events            10          orderId    7 days     FROM inventory-service
                                                              InventoryReservedEvent
                                                              InventoryUnavailableEvent
                                                              InventoryReleasedEvent
                                                              ReservationExpiredEvent

payment-events              20          orderId    7 days     FROM payment-service
                                                              PaymentSucceededEvent
                                                              PaymentFailedEvent
                                                              PaymentUnknownEvent
                                                              PaymentResolvedEvent
                                                              PaymentRefundedEvent

shipment-events             10          orderId    7 days     FROM shipment-service
                                                              ShipmentCreatedEvent
                                                              ShipmentDispatchedEvent
                                                              ShipmentDeliveredEvent

notification-events         5           userId     1 day      FROM order-events consumer
notification-dlq            3           userId     30 days    Failed notifications

loyalty-events              5           userId     7 days     FROM order-events consumer
warehouse-events            10          warehouseId 7 days   FROM order-events consumer
```

**All topics partitioned by orderId** — guarantees all events for the same order arrive in the same partition → total ordering per order.

**Consumer groups per topic:**

```
order-commands:   inventory-service (group: inventory-cmds)
                  payment-service   (group: payment-cmds)
order-events:     notification-service  (group: notifications)
                  loyalty-service       (group: loyalty)
                  warehouse-service     (group: warehouse)
                  order-query-service   (group: order-read-model)  → MongoDB
inventory-events: order-service     (group: order-saga)
payment-events:   order-service     (group: order-saga)
shipment-events:  order-service     (group: order-saga)
                  notification-service  (group: notifications)
```

---

## 11. Implementation Code

### 11.1 Order Service — Saga Orchestrator

```java
@RestController
@RequestMapping("/api/v1/orders")
@RequiredArgsConstructor
public class OrderController {

    private final OrderCommandService orderCommandService;
    private final OrderQueryService orderQueryService;

    @PostMapping
    @ResponseStatus(HttpStatus.ACCEPTED)  // 202: accepted, processing async
    public OrderCreatedResponse placeOrder(
            @RequestBody @Valid PlaceOrderRequest request,
            @RequestHeader("Authorization") String token,
            @RequestHeader("Idempotency-Key") String idempotencyKey) {

        String userId = jwtService.extractUserId(token);
        return orderCommandService.placeOrder(request, userId, idempotencyKey);
    }

    @GetMapping("/{orderId}")
    public OrderResponse getOrder(@PathVariable String orderId,
                                  @RequestHeader("Authorization") String token) {
        return orderQueryService.getOrder(orderId, jwtService.extractUserId(token));
    }

    @PostMapping("/{orderId}/cancel")
    public void cancelOrder(@PathVariable String orderId,
                            @RequestHeader("Authorization") String token,
                            @RequestBody CancelOrderRequest request) {
        orderCommandService.cancelOrder(orderId, jwtService.extractUserId(token),
            request.getReason());
    }
}

@Service
@RequiredArgsConstructor
@Slf4j
public class OrderCommandService {

    private final OrderRepository orderRepo;
    private final SagaStateRepository sagaRepo;
    private final OutboxEventRepository outboxRepo;
    private final StringRedisTemplate redis;

    @Transactional
    public OrderCreatedResponse placeOrder(PlaceOrderRequest req, String userId, String idemKey) {
        // 1. Idempotency check
        if (redis.hasKey("idempotency:order:" + idemKey)) {
            String cached = redis.opsForValue().get("idempotency:order:" + idemKey);
            return deserialize(cached, OrderCreatedResponse.class);
        }
        if (orderRepo.existsByIdempotencyKey(idemKey)) {
            Order existing = orderRepo.findByIdempotencyKey(idemKey);
            return OrderCreatedResponse.of(existing);
        }

        // 2. Validate items synchronously (fast check — no reservation yet)
        validateItems(req.getItems());  // throws if item discontinued / address invalid

        // 3. Create order in PENDING state
        Order order = Order.builder()
            .userId(userId)
            .status(OrderStatus.PENDING)
            .items(buildOrderItems(req.getItems()))
            .totalCents(calculateTotal(req.getItems()))
            .shippingAddressId(req.getShippingAddressId())
            .idempotencyKey(idemKey)
            .version(0)
            .build();
        order = orderRepo.save(order);

        // 4. Create saga state
        SagaState saga = SagaState.builder()
            .sagaId(order.getId())
            .currentStep("INVENTORY_RESERVING")
            .status(SagaStatus.IN_PROGRESS)
            .stepsLog(List.of(SagaStep.of("ORDER_CREATED", Instant.now())))
            .build();
        sagaRepo.save(saga);

        // 5. Write to outbox (atomic with order creation)
        outboxRepo.save(OutboxEvent.builder()
            .aggregateId(order.getId())
            .aggregateType("ORDER")
            .topic("order-commands")
            .partitionKey(order.getId().toString())
            .eventType("ReserveInventoryCommand")
            .payload(serialize(new ReserveInventoryCommand(
                order.getId(),
                order.getItems().stream()
                    .map(i -> new ReservationItem(i.getSkuId(), i.getQuantity()))
                    .collect(Collectors.toList()),
                Instant.now().plusSeconds(900)  // 15-minute expiry
            )))
            .build());

        // Outbox relay will publish to Kafka asynchronously

        // 6. Cache idempotency key
        OrderCreatedResponse response = OrderCreatedResponse.of(order);
        redis.opsForValue().set("idempotency:order:" + idemKey,
            serialize(response), Duration.ofHours(24));

        log.info("Order {} created for userId={}", order.getId(), userId);
        return response;
    }

    // ── Saga event handlers ──────────────────────────────────────────────

    @KafkaListener(topics = "inventory-events", groupId = "order-saga")
    @Transactional
    public void onInventoryEvent(InventoryEvent event) {
        SagaState saga = sagaRepo.findById(event.getOrderId()).orElseThrow();
        Order order = orderRepo.findById(event.getOrderId()).orElseThrow();

        switch (event.getEventType()) {
            case "InventoryReservedEvent" -> {
                // Move to payment step
                order.setStatus(OrderStatus.PAYMENT_PENDING);
                saga.setCurrentStep("PAYMENT_CHARGING");
                saga.addStep("INVENTORY_RESERVED");
                orderRepo.save(order);
                sagaRepo.save(saga);

                // Publish PaymentChargeCommand via outbox
                outboxRepo.save(OutboxEvent.of("order-commands", "PaymentChargeCommand",
                    order.getId(), new PaymentChargeCommand(order.getId(), order.getTotalCents(),
                        order.getUserId())));
            }
            case "InventoryUnavailableEvent" -> {
                order.setStatus(OrderStatus.REJECTED);
                saga.setStatus(SagaStatus.FAILED);
                saga.addStep("INVENTORY_UNAVAILABLE");
                orderRepo.save(order);
                sagaRepo.save(saga);

                // Publish OrderRejectedEvent via outbox
                outboxRepo.save(OutboxEvent.of("order-events", "OrderRejectedEvent",
                    order.getId(), new OrderRejectedEvent(order.getId(), "OUT_OF_STOCK")));
            }
            case "InventoryReleasedEvent" -> {
                // Compensation complete after payment failure
                order.setStatus(OrderStatus.CANCELLED);
                saga.setStatus(SagaStatus.FAILED);
                saga.addStep("INVENTORY_RELEASED");
                orderRepo.save(order);
                sagaRepo.save(saga);

                outboxRepo.save(OutboxEvent.of("order-events", "OrderCancelledEvent",
                    order.getId(), new OrderCancelledEvent(order.getId(), "PAYMENT_FAILED")));
            }
            case "ReservationExpiredEvent" -> {
                // 15-minute timeout fired before payment
                order.setStatus(OrderStatus.CANCELLED);
                saga.setStatus(SagaStatus.FAILED);
                saga.addStep("RESERVATION_EXPIRED");
                orderRepo.save(order);
                sagaRepo.save(saga);

                outboxRepo.save(OutboxEvent.of("order-events", "OrderCancelledEvent",
                    order.getId(), new OrderCancelledEvent(order.getId(), "RESERVATION_EXPIRED")));
            }
        }
    }

    @KafkaListener(topics = "payment-events", groupId = "order-saga")
    @Transactional
    public void onPaymentEvent(PaymentEvent event) {
        SagaState saga = sagaRepo.findById(event.getOrderId()).orElseThrow();
        Order order = orderRepo.findById(event.getOrderId()).orElseThrow();

        switch (event.getEventType()) {
            case "PaymentSucceededEvent" -> {
                // Confirm order, create shipment, release reservation
                order.setStatus(OrderStatus.CONFIRMED);
                order.setConfirmedAt(Instant.now());
                saga.setCurrentStep("ORDER_CONFIRMING");
                saga.addStep("PAYMENT_SUCCEEDED");
                orderRepo.save(order);
                sagaRepo.save(saga);

                // Confirm inventory (reserve → confirmed)
                outboxRepo.save(OutboxEvent.of("order-commands", "ConfirmInventoryCommand",
                    order.getId(), new ConfirmInventoryCommand(order.getId())));

                // Notify downstream (choreography)
                outboxRepo.save(OutboxEvent.of("order-events", "OrderConfirmedEvent",
                    order.getId(), new OrderConfirmedEvent(order)));
            }
            case "PaymentFailedEvent" -> {
                // Trigger compensation: release inventory
                order.setStatus(OrderStatus.PAYMENT_FAILED);
                saga.setStatus(SagaStatus.COMPENSATING);
                saga.setCurrentStep("INVENTORY_RELEASING");
                saga.addStep("PAYMENT_FAILED");
                orderRepo.save(order);
                sagaRepo.save(saga);

                outboxRepo.save(OutboxEvent.of("order-commands", "ReleaseInventoryCommand",
                    order.getId(), new ReleaseInventoryCommand(order.getId())));
            }
        }
    }
}
```

### 11.2 Inventory Service — Reservation with TTL

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class InventoryService {

    private final InventoryRepository inventoryRepo;
    private final ReservationRepository reservationRepo;
    private final StringRedisTemplate redis;
    private final RedisScript<List<Long>> reserveScript;
    private final OutboxEventRepository outboxRepo;

    @KafkaListener(topics = "order-commands", groupId = "inventory-cmds")
    @Transactional
    public void handleCommand(InventoryCommand command) {
        switch (command.getCommandType()) {
            case "ReserveInventoryCommand" -> reserveInventory((ReserveInventoryCommand) command);
            case "ReleaseInventoryCommand" -> releaseInventory((ReleaseInventoryCommand) command);
            case "ConfirmInventoryCommand" -> confirmInventory((ConfirmInventoryCommand) command);
        }
    }

    private void reserveInventory(ReserveInventoryCommand cmd) {
        String orderId = cmd.getOrderId().toString();

        // Idempotency: already reserved for this order?
        if (reservationRepo.existsByOrderIdAndStatus(cmd.getOrderId(), ReservationStatus.ACTIVE)) {
            log.info("Reservation already exists for orderId={}", orderId);
            publishInventoryReservedEvent(cmd.getOrderId());
            return;
        }

        List<ReservationItem> failedItems = new ArrayList<>();

        for (ReservationItem item : cmd.getItems()) {
            String availKey = "inventory:" + item.getSkuId() + ":available";

            // Atomic Lua: check available, then decrement + set reservation
            String reservationId = UUID.randomUUID().toString();
            String reservationData = serialize(Map.of(
                "orderId", orderId,
                "skuId", item.getSkuId(),
                "quantity", item.getQuantity(),
                "expiresAt", cmd.getExpiresAt().toString()
            ));

            List<Long> result = redis.execute(reserveScript,
                List.of(availKey, "reservation:" + reservationId),
                String.valueOf(item.getQuantity()),
                "900",  // 15-minute TTL
                reservationData
            );

            if (result.get(0) == -1) {
                failedItems.add(item);  // insufficient stock
                log.warn("Insufficient stock for skuId={}, orderId={}", item.getSkuId(), orderId);
            }
        }

        if (!failedItems.isEmpty()) {
            // Rollback any successful reservations in this batch
            rollbackPartialReservations(cmd.getOrderId());
            outboxRepo.save(OutboxEvent.of("inventory-events", "InventoryUnavailableEvent",
                cmd.getOrderId(), new InventoryUnavailableEvent(cmd.getOrderId(), failedItems)));
            return;
        }

        // All items reserved successfully — persist to PostgreSQL
        for (ReservationItem item : cmd.getItems()) {
            Reservation reservation = Reservation.builder()
                .orderId(cmd.getOrderId())
                .skuId(item.getSkuId())
                .quantity(item.getQuantity())
                .status(ReservationStatus.ACTIVE)
                .expiresAt(cmd.getExpiresAt())
                .build();
            reservationRepo.save(reservation);

            // Also update PostgreSQL inventory table
            int updated = inventoryRepo.decrementAvailableAndIncrementReserved(
                item.getSkuId(), item.getQuantity());
            if (updated == 0) throw new ConcurrentModificationException("Inventory version conflict");
        }

        outboxRepo.save(OutboxEvent.of("inventory-events", "InventoryReservedEvent",
            cmd.getOrderId(), new InventoryReservedEvent(cmd.getOrderId())));
        log.info("Inventory reserved for orderId={}", orderId);
    }

    private void releaseInventory(ReleaseInventoryCommand cmd) {
        List<Reservation> reservations = reservationRepo
            .findByOrderIdAndStatus(cmd.getOrderId(), ReservationStatus.ACTIVE);

        for (Reservation res : reservations) {
            // Restore Redis availability counter
            redis.opsForValue().increment(
                "inventory:" + res.getSkuId() + ":available", res.getQuantity());

            // Remove reservation key from Redis
            redis.delete("reservation:" + res.getId());

            // Update PostgreSQL
            res.setStatus(ReservationStatus.RELEASED);
            res.setReleasedAt(Instant.now());
            reservationRepo.save(res);

            inventoryRepo.decrementReservedAndIncrementAvailable(
                res.getSkuId(), res.getQuantity());
        }

        outboxRepo.save(OutboxEvent.of("inventory-events", "InventoryReleasedEvent",
            cmd.getOrderId(), new InventoryReleasedEvent(cmd.getOrderId())));
        log.info("Inventory released for orderId={}", cmd.getOrderId());
    }

    private void confirmInventory(ConfirmInventoryCommand cmd) {
        List<Reservation> reservations = reservationRepo
            .findByOrderIdAndStatus(cmd.getOrderId(), ReservationStatus.ACTIVE);

        for (Reservation res : reservations) {
            // Reservation confirmed → permanently deduct from physical stock
            res.setStatus(ReservationStatus.CONFIRMED);
            reservationRepo.save(res);

            // Remove Redis reservation key (no longer needed)
            redis.delete("reservation:" + res.getId());

            // PostgreSQL: decrement both reserved_qty and physical_qty
            inventoryRepo.confirmReservation(res.getSkuId(), res.getQuantity());
        }

        log.info("Inventory confirmed for orderId={}", cmd.getOrderId());
    }
}
```

### 11.3 Payment Service — Exactly-Once with Timeout Handling

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class PaymentService {

    private final PaymentRepository paymentRepo;
    private final OutboxEventRepository outboxRepo;
    private final PaymentGatewayClient gateway;
    private final StringRedisTemplate redis;

    @KafkaListener(topics = "order-commands", groupId = "payment-cmds")
    @Transactional
    public void handlePaymentCommand(PaymentChargeCommand cmd) {
        String orderId = cmd.getOrderId().toString();
        String idemKey = "payment:" + orderId;

        // Idempotency check
        Optional<Payment> existing = paymentRepo.findByIdempotencyKey(idemKey);
        if (existing.isPresent()) {
            Payment p = existing.get();
            if (p.getStatus() == PaymentStatus.SUCCEEDED) {
                publishPaymentSucceeded(p);
            } else if (p.getStatus() == PaymentStatus.FAILED) {
                publishPaymentFailed(p);
            }
            // UNKNOWN or PROCESSING: let the resolution job handle it
            return;
        }

        // Acquire lock to prevent concurrent payment attempts
        String lockKey = "payment:lock:" + orderId;
        Boolean acquired = redis.opsForValue()
            .setIfAbsent(lockKey, "1", Duration.ofSeconds(60));
        if (Boolean.FALSE.equals(acquired)) {
            log.warn("Payment already in progress for orderId={}", orderId);
            return;  // will retry via Kafka redelivery
        }

        // Create payment record in PROCESSING state
        Payment payment = Payment.builder()
            .orderId(cmd.getOrderId())
            .idempotencyKey(idemKey)
            .amountCents(cmd.getAmountCents())
            .status(PaymentStatus.PROCESSING)
            .gateway(resolveGateway(cmd))
            .build();
        payment = paymentRepo.save(payment);

        String paymentId = payment.getId().toString();
        redis.delete(lockKey);  // release lock after creating record

        // Execute charge asynchronously (non-blocking Kafka consumer thread)
        final Payment finalPayment = payment;
        paymentExecutor.execute(() -> executeCharge(finalPayment, cmd));
    }

    @Async("paymentExecutor")
    public void executeCharge(Payment payment, PaymentChargeCommand cmd) {
        try {
            GatewayResponse response = gateway.charge(
                ChargeRequest.builder()
                    .amountCents(cmd.getAmountCents())
                    .userId(cmd.getUserId())
                    .idempotencyKey(payment.getIdempotencyKey())
                    .orderId(cmd.getOrderId().toString())
                    .build(),
                Duration.ofSeconds(35)  // hard timeout
            );

            processGatewayResponse(payment, response);

        } catch (GatewayTimeoutException e) {
            log.warn("Gateway timeout for payment {}", payment.getId());
            handleTimeout(payment, e.getGatewayAttemptId());
        } catch (Exception e) {
            log.error("Unexpected error charging payment {}", payment.getId(), e);
            markPaymentFailed(payment, "INTERNAL_ERROR", e.getMessage());
        }
    }

    @Transactional
    private void processGatewayResponse(Payment payment, GatewayResponse response) {
        if (response.isSuccess()) {
            payment.setStatus(PaymentStatus.SUCCEEDED);
            payment.setGatewayTxnId(response.getTransactionId());
            payment.setCompletedAt(Instant.now());
            paymentRepo.save(payment);

            outboxRepo.save(OutboxEvent.of("payment-events", "PaymentSucceededEvent",
                payment.getOrderId(), new PaymentSucceededEvent(payment.getOrderId(),
                    payment.getId(), payment.getAmountCents())));
        } else {
            markPaymentFailed(payment, response.getFailureCode(), response.getFailureMessage());
        }
    }

    @Transactional
    private void handleTimeout(Payment payment, String gatewayAttemptId) {
        payment.setStatus(PaymentStatus.UNKNOWN);
        payment.setGatewayAttemptId(gatewayAttemptId);
        payment.setResolutionAttempts(0);
        payment.setNextResolutionAt(Instant.now().plusSeconds(5));
        paymentRepo.save(payment);
        // No Kafka event published yet — resolution job will handle this
    }

    @Scheduled(fixedDelay = 5000)
    public void resolveUnknownPayments() {
        List<Payment> unknowns = paymentRepo
            .findByStatusAndNextResolutionAtBefore(PaymentStatus.UNKNOWN, Instant.now());

        for (Payment payment : unknowns) {
            try {
                GatewayStatusResponse status =
                    gateway.queryStatus(payment.getGatewayAttemptId());

                transactionTemplate.execute(txStatus -> {
                    switch (status.getStatus()) {
                        case "SUCCEEDED" -> processGatewayResponse(payment,
                            GatewayResponse.success(status.getTransactionId()));
                        case "FAILED" -> markPaymentFailed(payment,
                            status.getFailureCode(), status.getFailureMessage());
                        default -> scheduleNextPoll(payment);
                    }
                    return null;
                });
            } catch (Exception e) {
                log.error("Resolution poll failed for {}", payment.getId(), e);
                scheduleNextPoll(payment);
            }
        }
    }

    @Transactional
    private void scheduleNextPoll(Payment payment) {
        int attempts = payment.getResolutionAttempts() + 1;
        payment.setResolutionAttempts(attempts);

        if (attempts >= 5) {
            payment.setStatus(PaymentStatus.TIMEOUT_UNRESOLVED);
            payment.setNextResolutionAt(null);
            paymentRepo.save(payment);
            log.error("Payment {} unresolved after 5 polls — needs manual review", payment.getId());
            // Alert ops team via PagerDuty/email
            outboxRepo.save(OutboxEvent.of("payment-events", "PaymentTimeoutUnresolvedEvent",
                payment.getOrderId(), new PaymentTimeoutUnresolvedEvent(payment.getOrderId())));
        } else {
            long[] delays = {5, 10, 15, 30, 60};
            payment.setNextResolutionAt(Instant.now().plusSeconds(delays[attempts - 1]));
            paymentRepo.save(payment);
        }
    }
}
```

### 11.4 Notification Service — Resilient Fan-out

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class NotificationService {

    private final NotificationLogRepository notifRepo;
    private final OutboxEventRepository outboxRepo;
    private final EmailClient emailClient;
    private final SmsClient smsClient;

    @KafkaListener(topics = "order-events", groupId = "notifications")
    @Transactional
    public void handleOrderEvent(OrderEvent event) {
        switch (event.getEventType()) {
            case "OrderConfirmedEvent"  -> scheduleNotifications(event, NotificationType.ORDER_CONFIRMED);
            case "OrderCancelledEvent" -> scheduleNotifications(event, NotificationType.ORDER_CANCELLED);
            case "OrderRejectedEvent"  -> scheduleNotifications(event, NotificationType.ORDER_REJECTED);
        }
    }

    @KafkaListener(topics = "shipment-events", groupId = "notifications")
    @Transactional
    public void handleShipmentEvent(ShipmentEvent event) {
        if ("ShipmentDispatchedEvent".equals(event.getEventType())) {
            scheduleNotifications(event, NotificationType.ORDER_SHIPPED);
        }
    }

    private void scheduleNotifications(Object event, NotificationType type) {
        String orderId = extractOrderId(event);
        String userId  = extractUserId(event);

        // Deduplicate: check if notification already scheduled/sent
        if (notifRepo.existsByOrderIdAndType(orderId, type)) {
            log.info("Notification already queued for orderId={} type={}", orderId, type);
            return;
        }

        // Log notification intent (atomic with outbox events)
        NotificationLog log = NotificationLog.builder()
            .orderId(orderId)
            .userId(userId)
            .type(type)
            .channels(List.of("EMAIL", "SMS"))
            .status(NotificationStatus.PENDING)
            .build();
        notifRepo.save(log);

        // Schedule email via outbox
        outboxRepo.save(OutboxEvent.of("notification-events", "SendEmailCommand",
            orderId, buildEmailPayload(event, type, userId)));

        // Schedule SMS via outbox
        outboxRepo.save(OutboxEvent.of("notification-events", "SendSmsCommand",
            orderId, buildSmsPayload(event, type, userId)));
    }

    @KafkaListener(topics = "notification-events", groupId = "notification-sender")
    public void sendNotification(NotificationCommand cmd) {
        int attempt = 0;
        int[] delaySeconds = {0, 30, 300, 1800, 7200};  // 0s, 30s, 5m, 30m, 2h

        for (int maxAttempts = delaySeconds.length; attempt < maxAttempts; attempt++) {
            try {
                if (attempt > 0) Thread.sleep(delaySeconds[attempt] * 1000L);

                if ("SendEmailCommand".equals(cmd.getType())) {
                    emailClient.send(cmd.getTo(), cmd.getSubject(), cmd.getBody());
                } else {
                    smsClient.send(cmd.getTo(), cmd.getMessage());
                }

                // Success: update notification log
                notifRepo.markChannelSent(cmd.getOrderId(), cmd.getType(), cmd.getChannel());
                return;

            } catch (Exception e) {
                log.warn("Notification attempt {}/{} failed for orderId={}: {}",
                    attempt + 1, maxAttempts, cmd.getOrderId(), e.getMessage());
            }
        }

        // All retries exhausted — move to DLQ
        log.error("Notification failed after all retries for orderId={}", cmd.getOrderId());
        notifRepo.markChannelFailed(cmd.getOrderId(), cmd.getType(), cmd.getChannel());
        // Kafka consumer will NOT commit offset → message goes to DLQ topic
        throw new NotificationPermanentlyFailedException(cmd.getOrderId());
    }
}
```

### 11.5 Reservation Expiry Worker

```java
// Listens to Redis keyspace expiry events for reservation keys
@Component
@RequiredArgsConstructor
@Slf4j
public class ReservationExpiryWorker implements MessageListener {

    private final InventoryService inventoryService;
    private final ReservationRepository reservationRepo;
    private final OutboxEventRepository outboxRepo;

    @Override
    @Transactional
    public void onMessage(Message message, byte[] pattern) {
        String expiredKey = new String(message.getBody());

        // Only handle reservation key expirations
        if (!expiredKey.startsWith("reservation:order:")) return;

        String orderId = expiredKey.substring("reservation:order:".length());
        log.info("Reservation expired for orderId={}", orderId);

        // Check if already processed (Redis keyspace events can duplicate)
        List<Reservation> activeReservations = reservationRepo
            .findByOrderIdAndStatus(UUID.fromString(orderId), ReservationStatus.ACTIVE);

        if (activeReservations.isEmpty()) return;  // already handled

        // Restore inventory counters
        for (Reservation res : activeReservations) {
            redis.opsForValue().increment(
                "inventory:" + res.getSkuId() + ":available", res.getQuantity());

            res.setStatus(ReservationStatus.EXPIRED);
            res.setReleasedAt(Instant.now());
            reservationRepo.save(res);

            inventoryRepo.decrementReservedAndIncrementAvailable(
                res.getSkuId(), res.getQuantity());
        }

        // Notify order saga
        outboxRepo.save(OutboxEvent.of("inventory-events", "ReservationExpiredEvent",
            UUID.fromString(orderId), new ReservationExpiredEvent(UUID.fromString(orderId))));
    }

    // Safety net: scheduled DB-based expiry check (catches missed Redis events)
    @Scheduled(cron = "0 * * * * *")  // every minute
    @Transactional
    public void expireStaleReservations() {
        List<Reservation> stale = reservationRepo
            .findByStatusAndExpiresAtBefore(ReservationStatus.ACTIVE, Instant.now());

        for (Reservation res : stale) {
            log.warn("Safety-net expiry triggered for reservationId={}", res.getId());
            // Same logic as above
            redis.opsForValue().increment(
                "inventory:" + res.getSkuId() + ":available", res.getQuantity());
            res.setStatus(ReservationStatus.EXPIRED);
            res.setReleasedAt(Instant.now());
            reservationRepo.save(res);
        }
    }
}
```

---

## 12. Failure Scenarios & Mitigations

### Scenario 1: Order-Service Crashes Mid-Saga

```
Problem: order-service processes InventoryReservedEvent, writes DB, crashes
         before writing to outbox → PaymentChargeCommand never published
         Order stuck in INVENTORY_PENDING forever

Mitigation:
  1. Outbox pattern: DB write (order status + outbox event) is ONE transaction
     Either both committed or both rolled back
     If crash before commit: Kafka consumer retries (at-least-once delivery)
     inventory-service republishes InventoryReservedEvent → order-service handles it again
     Idempotency check: "InventoryReserved" already in steps_log → skip duplicate

  2. Saga timeout monitor (every 5 minutes):
     SELECT * FROM saga_state
     WHERE status = 'IN_PROGRESS'
       AND updated_at < NOW() - INTERVAL '10 minutes'
     → Alert + manual investigation OR auto-advance if safe
```

### Scenario 2: Inventory Double-Reserve (Same Order, Two Kafka Deliveries)

```
Problem: InventoryReservedEvent published twice (Kafka at-least-once)
         inventory-service processes it twice → reserves double the stock

Mitigation:
  1. Idempotency at handler entry:
     "Does active reservation exist for orderId X?" → YES → skip + republish success event
  2. PostgreSQL UNIQUE constraint on (order_id, sku_id) in reservations table
     Second INSERT violates constraint → exception → transaction rolled back → no double reserve
  3. Lua script atomic check: Redis key "reservation:order:{orderId}" set with SETNX
     Second attempt sees key exists → returns immediately
```

### Scenario 3: Payment Gateway Partial Failure (Network Asymmetry)

```
Problem: Gateway charges card but response lost in transit
         payment-service marks payment as UNKNOWN
         Resolution polls show PENDING for >5 minutes

Mitigation:
  1. Timeout resolution with 5 retry polls (5s, 10s, 15s, 30s, 60s intervals)
  2. After 5 polls: TIMEOUT_UNRESOLVED state
     → Inventory released (safety first — customer not charged is better than oversell)
     → Payment moved to manual review queue
     → Customer notified: "Order under review, we'll confirm within 1 hour"
  3. Operations team resolves via admin portal:
     - If gateway confirms charge: refund → customer gets email "order issue resolved"
     - If gateway confirms no charge: reprocess order or cancel cleanly
```

### Scenario 4: Kafka Consumer Group Rebalance Mid-Saga

```
Problem: Consumer group rebalances while processing inventory-reserved event
         New consumer gets same message and starts processing again
         Two concurrent payment charges attempted

Mitigation:
  1. Payment idempotency key: "payment:{orderId}" → second charge returns cached result
  2. Payment lock in Redis: "payment:lock:{orderId}" with 60s TTL
     Second attempt finds lock → returns immediately
  3. Saga saga:lock:{orderId} in Redis: only one saga step runs at a time
  4. PostgreSQL UNIQUE constraint on payments.idempotency_key
```

### Scenario 5: Warehouse Notification Service Down

```
Problem: Warehouse service is down; OrderConfirmedEvent sits in Kafka
         Warehouse never receives fulfillment task
         Order confirmed from customer perspective, but never fulfilled

Mitigation:
  1. Kafka retains messages (7 days retention on order-events)
  2. Warehouse consumer group will catch up when service recovers
  3. Monitor: consumer lag alert if warehouse-service lag > 100 orders
  4. Escalation: if warehouse consumer lag > 1,000 OR age > 2 hours:
     → PagerDuty alert to operations
     → Manual fulfillment task creation if needed
  5. SLA: "warehouse notified within 1 hour of order confirmation" → tracked as KPI
```

---

## 13. Scaling Strategy

```
Service                  Scale Model    Target Capacity     Bottleneck
───────────────────────────────────────────────────────────────────────────
order-service            Horizontal     100 pods            Saga state DB writes
inventory-service        Horizontal     20 pods             Redis atomic ops
payment-service          Horizontal     50 pods             Payment gateway API limits
notification-service     Horizontal     10 pods             SendGrid/SMS rate limits
loyalty-service          Horizontal     5 pods              PostgreSQL writes
warehouse-service        Horizontal     5 pods              WMS API rate limits

PostgreSQL               1P + 2RR       10K writes/sec      Orders DB
Redis                    Cluster (6)    500K ops/sec         Inventory counters
Kafka                    6 brokers      5M msgs/sec          Not a bottleneck here
```

**Throughput at 10K orders/day (baseline):**

```
7 orders/sec × 8 pipeline steps = 56 Kafka events/sec → trivial
35 orders/sec peak × 8 steps   = 280 Kafka events/sec → trivial

Redis inventory ops:
  35 DECRBY/sec for reservations → 35 ops/sec → trivially handled

PostgreSQL writes:
  35 orders/sec × 5 writes each = 175 writes/sec → well within capacity

Payment gateway:
  35 concurrent charges/sec → check Stripe/Braintree concurrency limits
  Typical limit: 100 concurrent → fine at 35/sec
  Each charge holds thread for 0–30s → async pattern (Kafka + executor) prevents blocking
```

**Payment executor thread pool sizing:**

```java
@Bean(name = "paymentExecutor")
public Executor paymentExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(50);       // 50 concurrent payment charges
    executor.setMaxPoolSize(100);       // burst up to 100
    executor.setQueueCapacity(500);     // queue overflow
    executor.setThreadNamePrefix("payment-");
    executor.setRejectedExecutionHandler(new CallerRunsPolicy());
    executor.initialize();
    return executor;
}
// 50 threads × 30s max each = can handle 50 simultaneous gateway calls
// At 35 orders/sec average: needs 35 × avg_gateway_time (2s) = 70 threads
// MaxPoolSize=100 → sufficient headroom
```

---

## 14. Monitoring & Observability

### Key Metrics

```
Throughput:
  orders_placed_total{status}                → orders/min by result
  orders_confirmed_total                     → successful completions/min
  orders_cancelled_total{reason}             → cancellations by reason
  saga_step_duration_seconds{step}           → latency per pipeline step

Inventory:
  inventory_reservations_active              → current active holds
  inventory_reservation_expired_total        → timeouts per hour (alert if high)
  inventory_reserve_failed_total             → out-of-stock rejections

Payment:
  payment_attempts_total{status}             → success/failed/unknown/unresolved
  payment_gateway_latency_p99                → target <5s (alert if approaching 30s)
  payment_unknown_count                      → currently unresolved UNKNOWN payments
  payment_timeout_unresolved_total           → escalation needed

Notifications:
  notification_sent_total{channel, type}     → emails/SMS sent
  notification_failed_total{channel}         → delivery failures
  notification_dlq_size                      → dead letter queue depth (alert if > 0)

Saga:
  saga_in_progress_count                     → sagas currently running
  saga_stuck_count                           → sagas not advanced in > 10 min (ALERT)
  saga_compensation_total                    → rollbacks triggered

Kafka:
  kafka_consumer_lag{group, topic}           → target <100 for order-saga
                                             → alert if > 1000 for any consumer
```

### Alerts

```
CRITICAL:
  payment_timeout_unresolved_total > 0       → Manual review required immediately
  saga_stuck_count > 0                        → Order pipeline broken
  inventory_available < 0 for any SKU        → Oversell occurred! (should never happen)
  outbox_dead_lettered_count > 0             → Event lost, manual intervention
  kafka_consumer_lag{order-saga} > 500       → Order saga severely delayed

WARNING:
  payment_gateway_latency_p99 > 15s          → Gateway degraded, approaching timeout
  inventory_reservation_expired_rate > 10%   → Users not completing payment (UX issue)
  notification_failed_rate > 5%              → Email/SMS service degraded
  kafka_consumer_lag > 100                   → Any consumer falling behind
```

### Distributed Tracing (OpenTelemetry)

```
Trace spans for an order from placement to confirmation:

  POST /orders                               ~5ms   (sync validation + DB write)
  ├── validate_items                         ~3ms
  ├── create_order (PostgreSQL INSERT)       ~2ms
  └── write_outbox (PostgreSQL INSERT)       ~1ms

  [outbox relay]                             ~100ms (async, not blocking user)
  ├── Kafka publish: ReserveInventoryCommand

  [inventory-service]
  ├── check_redis_availability               ~1ms
  ├── lua_atomic_reserve                     ~1ms
  ├── postgres_write_reservation             ~3ms
  └── publish_InventoryReservedEvent         ~5ms

  [order-service saga step]                  ~10ms
  └── publish_PaymentChargeCommand

  [payment-service]
  ├── gateway_http_call                      ~2000ms (avg gateway)
  └── publish_PaymentSucceededEvent

  [order-service saga step]                  ~15ms
  └── publish_OrderConfirmedEvent

  Total (end-to-end, happy path): ~2.2 seconds
  Payment step dominates (gateway latency)
```

---

## 15. Summary Cheat Sheet

### The 6 Core Answers at a Glance

| Question | Answer | Key Technology |
|---|---|---|
| What microservices? | order (orchestrator), inventory, payment, shipment, notification, loyalty, warehouse | Saga orchestration |
| Sync or async? | Sync only for initial validation; async (Kafka) for all pipeline steps | Kafka events |
| Exactly-once payment? | 3 layers: client idempotency key → Redis lock + DB unique → gateway idempotency key | Redis SETNX + PostgreSQL UNIQUE |
| Inventory reservation timeout? | Redis key TTL (900s) → keyspace expiry event → release inventory + cancel order + DB scheduled safety net | Redis TTL + keyspace events |
| Email service down? | Email is non-blocking. OrderConfirmed publishes to Kafka. Notification-service retries independently. DLQ after 5 attempts. | Kafka at-least-once + DLQ |
| Payment gateway timeout? | Treat timeout as UNKNOWN (not failed). Poll gateway every 5-60s for resolution. After 5 polls: manual review + inventory release. | Scheduled polling + PaymentStatus enum |

### Golden Rules for Order Processing Pipelines

```
1.  Orders must be durable from the first write — persist before returning 202 Accepted
2.  Outbox Pattern — never dual-write to DB and Kafka; write DB row + outbox row in one transaction
3.  Saga orchestration — one service owns the workflow; others are stateless workers
4.  Idempotency everywhere — every handler must check "did I process this already?"
5.  Three idempotency layers for payments: client key → service lock → gateway key
6.  Timeout ≠ Failure — always poll for resolution before releasing inventory
7.  Async payment processing — 30s gateway calls must never block Kafka consumer threads
8.  Reservation TTL in Redis + scheduled DB sweep — two expiry mechanisms for safety
9.  Compensating transactions — every forward step must have a defined rollback
10. Non-blocking notifications — email/SMS failure must never affect order status
11. Partition Kafka topics by orderId — guarantees ordering of events per order
12. Partial index on outbox_events(published=false) — makes relay query O(1) not O(N)
```

### Technology Role Summary

```
PostgreSQL:   ★ Orders, order_items — transactional source of record
              ★ Inventory and reservations — ACID stock management
              ★ Payments — financial audit trail (never lose a payment record)
              ★ Saga state — workflow tracking and debugging
              ★ Outbox events — guaranteed event publishing (the key pattern)

Redis:        ★ Inventory available counters (atomic DECRBY — no oversell)
              ★ Reservation TTL keys (automatic 15-minute expiry)
              ★ Payment idempotency cache (24h fast lookup)
              ★ Payment/saga distributed locks (prevent concurrent processing)
              ★ Keyspace expiry notifications (drive reservation expiry workflow)

Kafka:        ★ Pipeline backbone — decouples all 8 pipeline steps
              ★ Commands (to services) and Events (from services) both use Kafka
              ★ Ordered per orderId — all events for one order in same partition
              ★ Consumer group isolation — each service processes independently
              ★ 7-day retention — replay capability for outage recovery
              ★ DLQ for permanently failed notifications

MongoDB:      ★ Order history (denormalized, read-optimized for customer view)
              ★ Audit trail — every state transition recorded immutably
              ★ Customer order timeline (my orders page)

Resilience4j: ★ Circuit breakers around payment gateway, inventory service
              ★ Retry with exponential backoff + jitter for transient failures
              ★ Bulkhead: payment executor thread pool isolated from main threads
              ★ Timeout enforcement on gateway calls (35s hard cap)
```
