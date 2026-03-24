# Flash Sale System — Deep Dive Design

> **Scenario:** 100 units of iPhone at 50% off. Sale starts at exactly 12:00 PM.
> 1 million concurrent users. First 100 successful payments win. No overselling.
> Users must know their queue position.

**Tech Stack:** Java 17 · Spring Boot 3 · PostgreSQL · MongoDB · Redis · Kafka · WebSocket

---

## Table of Contents

1. [Requirements & Constraints](#1-requirements--constraints)
2. [Capacity Estimation](#2-capacity-estimation)
3. [High-Level Architecture](#3-high-level-architecture)
4. [Core Design Questions Answered](#4-core-design-questions-answered)
   - 4.1 How do you prevent overselling?
   - 4.2 How do you handle 1M concurrent requests?
   - 4.3 Optimistic vs Pessimistic Locking?
   - 4.4 How do you implement the waiting queue?
   - 4.5 How do you ensure fairness (no bots)?
   - 4.6 What is your caching strategy?
5. [Microservices Breakdown](#5-microservices-breakdown)
6. [Database Design](#6-database-design)
7. [Redis Data Structures](#7-redis-data-structures)
8. [Kafka Event Flow](#8-kafka-event-flow)
9. [Implementation Code](#9-implementation-code)
   - 9.1 Flash Sale Admission (Rate Gate)
   - 9.2 Redis Inventory Decrement (Lua Script)
   - 9.3 Queue Service
   - 9.4 WebSocket Queue Position
   - 9.5 Order Service with Saga Pattern
   - 9.6 Payment Service
10. [Failure Scenarios & Mitigations](#10-failure-scenarios--mitigations)
11. [Scaling Strategy](#11-scaling-strategy)
12. [Monitoring & Observability](#12-monitoring--observability)
13. [Summary Cheat Sheet](#13-summary-cheat-sheet)

---

## 1. Requirements & Constraints

### Functional Requirements

- Flash sale for exactly **100 units** of a product at 50% discount
- Sale window: exact start time (12:00 PM), runs until all 100 units are sold
- Users must be placed in a **fair queue** — first-come, first-served
- Users can see their **real-time queue position**
- **Exactly 100 successful purchases** — no overselling, no underselling
- System must handle **1 million concurrent users** without crashing
- Failed payments must release inventory back to the queue (next user gets it)

### Non-Functional Requirements

- **Availability:** 99.99% during sale window
- **Latency:** Queue admission < 50ms; order confirmation < 2s
- **Consistency:** Strong for inventory (no overselling); eventual for queue position display
- **Idempotency:** Double-clicking "Buy" must not create two orders
- **Fairness:** Bots and scripts must not get preferential treatment

### Out of Scope

- Product catalog browsing (handled by a separate service)
- User authentication flows (JWT assumed pre-issued)
- Refund / return flows

---

## 2. Capacity Estimation

```
Users:            1,000,000 concurrent
Units available:  100
Sale duration:    ~seconds to minutes

Request Rate:
  Pre-sale (countdown):     ~500,000 req/s  (polling for start)
  At-sale (burst):         ~1,000,000 req/s (everyone clicks at once)
  Post-admission:          ~100 active orders being processed

Traffic math:
  1M users × avg 2 KB payload = 2 GB/s ingress (peak burst)
  Need ~500 API Gateway nodes at 4M req/s/node

Redis:
  Inventory counter:  1 key, O(1) atomic DECR
  Queue:              1 sorted set, up to 1M members
  Each member:        ~64 bytes → 64 MB for 1M users (trivial)
  Queue ops:          1M ZADD at sale start → needs pipeline/batching

PostgreSQL:
  Orders table:   100 rows written (tiny)
  Holds table:    up to 100 rows at a time

Bandwidth:
  WebSocket connections: 1M open sockets × 1 KB/s heartbeat = 1 GB/s
  → Need dedicated WebSocket cluster with sticky sessions
```

---

## 3. High-Level Architecture

```
                        ┌─────────────────────────────────────────┐
                        │           CLIENT LAYER                  │
                        │  Browser / Mobile  (1M concurrent)      │
                        │  - WebSocket for queue position         │
                        │  - REST for purchase attempts           │
                        └──────────────┬──────────────────────────┘
                                       │
                        ┌──────────────▼──────────────────────────┐
                        │           CDN / WAF                     │
                        │  Cloudflare / AWS WAF                   │
                        │  - DDoS protection                      │
                        │  - Bot fingerprinting                   │
                        │  - Rate limit per IP/user               │
                        └──────────────┬──────────────────────────┘
                                       │
                        ┌──────────────▼──────────────────────────┐
                        │         API GATEWAY CLUSTER             │
                        │  (Kong / Spring Cloud Gateway)          │
                        │  - JWT validation                       │
                        │  - Global rate limiting (Redis token)   │
                        │  - Route to services                    │
                        └────┬─────────────┬───────────┬──────────┘
                             │             │           │
              ┌──────────────▼──┐  ┌───────▼──────┐  ┌▼──────────────────┐
              │  ADMISSION SVC  │  │  QUEUE SVC   │  │   WEBSOCKET SVC   │
              │ (Flash Gate)    │  │  (Redis ZSET) │  │  (Position Push)  │
              │ - Token bucket  │  │  - Enqueue   │  │  - Kafka consumer │
              │ - Queue user    │  │  - Position  │  │  - SSE / WS push  │
              └──────┬──────────┘  └──────┬───────┘  └───────────────────┘
                     │                   │
              ┌──────▼──────────────────▼────────────────────────────────┐
              │                    KAFKA                                  │
              │  Topics: sale-admitted | order-created | payment-result   │
              │          inventory-reserved | order-confirmed             │
              └──────┬───────────────────────┬────────────────────────────┘
                     │                       │
        ┌────────────▼──────┐    ┌───────────▼──────────┐
        │   ORDER SERVICE   │    │   PAYMENT SERVICE    │
        │  - Create order   │    │  - Stripe / PayPal   │
        │  - Hold inventory │    │  - Idempotency key   │
        │  - Saga orchestr. │    │  - 5min timeout      │
        └────────────┬──────┘    └───────────┬──────────┘
                     │                       │
        ┌────────────▼──────────────────────▼──────────────────┐
        │              INVENTORY SERVICE                        │
        │  - Redis for real-time stock (source of truth)        │
        │  - PostgreSQL for committed orders                    │
        │  - Release hold on payment failure                    │
        └────────────────────────────────────────────────────────┘

Data Stores:
  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
  │    REDIS        │  │  POSTGRESQL     │  │    MONGODB      │
  │  - Inventory    │  │  - Orders       │  │  - Queue logs   │
  │  - Queue ZSET   │  │  - Payments     │  │  - Audit trail  │
  │  - Rate limits  │  │  - Saga state   │  │  - User events  │
  │  - Idempotency  │  │  - Flash sale   │  │  - Analytics    │
  └─────────────────┘  └─────────────────┘  └─────────────────┘
```

---

## 4. Core Design Questions Answered

---

### 4.1 How Do You Prevent Overselling?

**The golden rule: Redis is the single source of truth for available inventory during a flash sale.**

Never use a database `UPDATE SET stock = stock - 1 WHERE stock > 0` as the primary guard — under concurrent load, race conditions will oversell.

**Solution: Atomic Redis DECR with a Lua script**

```
┌─────────────────────────────────────────────────────────┐
│  Flash Sale Inventory in Redis                          │
│                                                         │
│  Key: "flash_sale:{saleId}:inventory"  →  Value: 100   │
│                                                         │
│  Lua Script (atomic):                                   │
│    local stock = redis.call('GET', KEYS[1])             │
│    if tonumber(stock) <= 0 then                         │
│      return -1  -- sold out                             │
│    end                                                  │
│    return redis.call('DECR', KEYS[1])  -- returns new   │
│                                                         │
│  Why Lua? Redis executes Lua atomically — no other      │
│  command runs between the GET and DECR.                 │
└─────────────────────────────────────────────────────────┘
```

**The two-phase commit for inventory:**

```
Phase 1 — Reserve (Redis):
  DECR flash_sale:123:inventory
  → If result >= 0: inventory reserved, proceed to order
  → If result < 0:  INCR back (undo), user gets "sold out"

Phase 2 — Confirm (PostgreSQL):
  On successful payment: INSERT into orders, mark as CONFIRMED
  On failed payment:     INCR Redis back (release inventory)
                         Mark order as CANCELLED in PostgreSQL
```

**Why not just use PostgreSQL?**

Under 1M concurrent requests, even a `SELECT FOR UPDATE` on PostgreSQL creates massive lock contention. A single Redis node handles ~100,000 atomic operations/second with sub-millisecond latency. PostgreSQL becomes the durable record-of-truth after Redis has already serialized the decision.

---

### 4.2 How Do You Handle 1M Concurrent Requests?

**Strategy: Shed load before it hits business logic. Only ~100 requests should ever reach the inventory decrement.**

```
Layer 1 — CDN/WAF (sheds ~80%)
  - Block known bot signatures, Tor exit nodes
  - Rate limit per IP: max 10 req/s during sale
  - Geographic throttling if needed
  - ~200,000 requests pass through

Layer 2 — API Gateway Token Bucket (sheds ~80% of remaining)
  - Global token bucket: 50,000 tokens, refill 10,000/s
  - Per-user token: 1 request allowed during sale window
  - "Already in queue" check: return cached position
  - ~40,000 requests pass through to Admission

Layer 3 — Admission Service (queues fairly, sheds overflow)
  - Only first N users per second are queued (N = configurable)
  - Beyond queue capacity → "Sale full, try next time"
  - Uses Redis ZADD with timestamp as score (microseconds)
  - Queue max size = 1,000 (100 slots + 900 reserve)
  - ~1,000 users enter queue

Layer 4 — Queue Worker (processes 100 purchases serially)
  - Pops from queue one at a time
  - Decrements inventory
  - Creates order, initiates payment
  - Only 100 reach payment
```

**Token Bucket in Redis for global rate limiting:**

```
Key: "rate_limit:global:flash_sale"
Algorithm:
  tokens = GET key
  if tokens > 0:
    DECR key
    allow request
  else:
    reject with 429 Too Many Requests

Key: "rate_limit:user:{userId}:flash_sale"
  Allows exactly 1 admission attempt per user
  TTL = sale window duration
```

---

### 4.3 Optimistic vs Pessimistic Locking?

**Answer: Neither for inventory — use Redis atomic operations. Use Optimistic Locking for order state transitions in PostgreSQL.**

| Approach | Where | Why |
|---|---|---|
| Redis DECR (atomic) | Inventory count | O(1), no locks, no contention |
| Optimistic Locking | Order status updates | Low contention, avoid deadlocks |
| Pessimistic Locking | Never for inventory | Creates bottleneck under 1M users |
| Database transactions | Payment confirmation | ACID needed for financial data |

**Optimistic Locking for Order State in PostgreSQL:**

```sql
-- Order table has a version column
UPDATE orders
SET status = 'CONFIRMED', version = version + 1
WHERE order_id = ?
  AND version = ?       -- fails if another thread already updated
  AND status = 'PENDING'
```

If the update affects 0 rows → someone else changed the order → retry or raise conflict. This prevents double-confirmation without holding locks.

**Why not pessimistic locking for inventory?**

With 1M users hitting the same row:
- Thread 1 acquires lock → 999,999 threads wait
- Lock held for 50ms (DB write) → queue of waiting threads grows
- Lock release → thundering herd → DB CPU spikes to 100%
- Result: cascading timeouts and system crash

Redis atomic DECR has no waiting threads — the operation completes in <1ms with no contention.

---

### 4.4 How Do You Implement the Waiting Queue?

**Data structure: Redis Sorted Set (ZSET)**

```
Key:   "flash_sale:{saleId}:queue"
Score: Unix timestamp in microseconds (arrival time)
Value: userId

Commands:
  ZADD flash_sale:123:queue NX 1699800000123456 "user-abc"
    NX = only add if not already a member (no double-queueing)

  ZRANK flash_sale:123:queue "user-abc"
    → Returns 0-based position (position 0 = next to be served)

  ZPOPMIN flash_sale:123:queue
    → Removes and returns the user with the smallest timestamp
    → Used by queue worker to fetch next user

  ZCARD flash_sale:123:queue
    → Total queue length
```

**Queue Worker Flow:**

```
while (sale_active && inventory > 0):
  user = ZPOPMIN flash_sale:123:queue   [O(log N)]

  if user is null:
    WAIT 10ms, continue

  remaining = DECR flash_sale:123:inventory  [atomic]

  if remaining < 0:
    INCR flash_sale:123:inventory  [undo]
    ZADD flash_sale:123:queue user  [re-enqueue]
    break  -- sold out, stop processing

  create_order(user)
  notify_user(user, "proceed_to_payment")
  publish_event(Kafka, "order-created", orderId, userId)
```

**Queue Position Update to Users:**

```
Every 500ms (background job):
  For each user in queue (batch ZRANK calls):
    position = ZRANK flash_sale:123:queue userId
    Publish to Kafka topic: "queue-position-updates"

WebSocket Service consumes Kafka:
  Push position update to each connected user's socket
```

**Queue expiry:**

```
When user reaches front of queue:
  - Set payment timer: 5 minutes to complete payment
  - Key: "payment_timer:{orderId}" with TTL = 300s
  - If timer expires → release inventory back to Redis
  - Notify next user in queue
```

---

### 4.5 How Do You Ensure Fairness (No Bots)?

**Multi-layer bot prevention:**

**Layer 1 — Pre-Registration (hours before sale)**

```
24 hours before sale → Users must register interest
Registration generates:
  - A signed JWT with a unique nonce
  - Stored in Redis: "registered:{saleId}:{userId}" = nonce
  - Prevents last-second mass-registration by bots

Sale start:
  - Only pre-registered users are admitted to queue
  - Bots need to register accounts days in advance (expensive)
```

**Layer 2 — CAPTCHA at Registration**

```
reCAPTCHA v3 / hCaptcha at registration time (not at purchase)
  - Score-based, invisible to real users
  - Bots score < 0.5 → flagged
  - Flagged accounts → deprioritized in queue or blocked

At sale time: no CAPTCHA (adds latency, hurts real users)
```

**Layer 3 — Behavioral Fingerprinting**

```
Signals tracked per user (in MongoDB):
  - Mouse movement patterns (bots move in straight lines)
  - Scroll behavior
  - Time on page before clicking
  - Browser fingerprint (canvas, WebGL, fonts)
  - Request timing regularity (bots have perfect intervals)

Users with bot-like signatures:
  - Added to queue with a penalty score offset (+10 seconds)
  - Or rate-limited to 1 request per 10 seconds
```

**Layer 4 — One Purchase Per Account**

```
Redis check:
  Key: "purchased:{saleId}:{userId}"
  SETNX → if key exists, user already bought, reject

Backed by:
  PostgreSQL UNIQUE constraint on (sale_id, user_id) in orders table
  → Database-level enforcement as final guard
```

**Layer 5 — One Purchase Per Device / Payment Method**

```
Fingerprint the payment method (last 4 digits + bank)
Key: "purchased:{saleId}:{paymentFingerprint}"
Prevent family members sharing accounts to buy multiple units
```

---

### 4.6 What Is Your Caching Strategy?

**Different data has different caching needs during a flash sale:**

```
Data Type              Cache?    Store         TTL      Strategy
─────────────────────────────────────────────────────────────────
Inventory count        YES       Redis ONLY    N/A      Source of truth
                                               (no TTL)  (not just cache)

Flash sale metadata    YES       Redis + L1    1 hour   Cache-aside
(product, price, time)           Caffeine               Warm before sale

User queue position    YES       Redis ZSET    N/A      Live computed
                                               (no TTL)

User registration      YES       Redis         24 hrs   Write-through
status

Rate limit counters    YES       Redis         60s      Sliding window

Product details        YES       CDN +         10 min   Cache-aside
(images, description)            Redis                  Pre-warmed

Session / JWT tokens   YES       Redis         TTL from Token cache
                                               JWT exp

Order state            NO        PostgreSQL    n/a      Always DB
Payment records        NO        PostgreSQL    n/a      Always DB
```

**Pre-warming strategy (critical for flash sales):**

```
T-60 minutes: Warm CDN with product images and page HTML
T-30 minutes: Load flash sale metadata into Redis
              Key: "flash_sale:123:meta" → JSON blob
              Pre-build static "sale page" HTML, push to CDN edge

T-5 minutes:  Warm all application instances' L1 caches (Caffeine)
              Broadcast via Kafka: "pre-warm" event
              Each instance fetches and caches sale metadata

T=0:          All caches already hot
              No cache miss spike at sale start
              Inventory key already seeded: SET flash_sale:123:inventory 100
```

**Cache-aside implementation for sale metadata:**

```java
// Two-level cache: Caffeine (L1) → Redis (L2) → DB
public FlashSaleMeta getSaleMeta(String saleId) {
    // L1: local in-process cache (sub-millisecond)
    FlashSaleMeta meta = localCache.getIfPresent(saleId);
    if (meta != null) return meta;

    // L2: Redis (1ms)
    meta = redisTemplate.opsForValue().get("flash_sale:" + saleId + ":meta");
    if (meta != null) {
        localCache.put(saleId, meta);
        return meta;
    }

    // L3: Database (fallback only)
    meta = flashSaleRepository.findById(saleId).orElseThrow();
    redisTemplate.opsForValue().set("flash_sale:" + saleId + ":meta",
                                    meta, Duration.ofHours(1));
    localCache.put(saleId, meta);
    return meta;
}
```

---

## 5. Microservices Breakdown

```
Service                Responsibility                      Port
──────────────────────────────────────────────────────────────────
admission-service      Rate gate, queue entry, bot check   8081
queue-service          ZSET management, position calc      8082
order-service          Order creation, Saga orchestration  8083
inventory-service      Redis stock, PostgreSQL commit      8084
payment-service        Payment gateway integration         8085
notification-service   WebSocket, SSE, email, SMS          8086
flash-sale-admin       Sale config, start/stop controls    8087
analytics-service      MongoDB event logging, dashboard    8088
```

**Inter-service communication:**

- **Synchronous (REST):** Admission → Queue, Order → Inventory (reserve)
- **Asynchronous (Kafka):** All state change events
- **WebSocket:** Notification → Client (queue position, order status)

---

## 6. Database Design

### PostgreSQL — Transactional Data

```sql
-- Flash sale configuration
CREATE TABLE flash_sales (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id      UUID NOT NULL,
    total_units     INT NOT NULL,
    price_cents     BIGINT NOT NULL,
    original_price  BIGINT NOT NULL,
    starts_at       TIMESTAMPTZ NOT NULL,
    ends_at         TIMESTAMPTZ,
    status          VARCHAR(20) NOT NULL DEFAULT 'SCHEDULED',
                    -- SCHEDULED | ACTIVE | SOLD_OUT | ENDED
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- Orders table
CREATE TABLE orders (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sale_id         UUID NOT NULL REFERENCES flash_sales(id),
    user_id         UUID NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'PENDING',
                    -- PENDING | PAYMENT_INITIATED | CONFIRMED | CANCELLED | REFUNDED
    price_cents     BIGINT NOT NULL,
    idempotency_key VARCHAR(64) UNIQUE NOT NULL,
    version         INT NOT NULL DEFAULT 0,         -- optimistic lock
    payment_expires_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    confirmed_at    TIMESTAMPTZ,

    CONSTRAINT one_per_user_per_sale UNIQUE (sale_id, user_id)
);

CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_orders_sale_status ON orders(sale_id, status);

-- Saga state table (for distributed transaction tracking)
CREATE TABLE saga_state (
    saga_id         UUID PRIMARY KEY,
    order_id        UUID NOT NULL REFERENCES orders(id),
    current_step    VARCHAR(50) NOT NULL,
    -- INVENTORY_RESERVED | PAYMENT_INITIATED | COMPLETED | COMPENSATING | FAILED
    steps_completed JSONB NOT NULL DEFAULT '[]',
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW()
);

-- Payment records
CREATE TABLE payments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id        UUID NOT NULL REFERENCES orders(id),
    gateway         VARCHAR(20) NOT NULL,           -- STRIPE | PAYPAL
    gateway_txn_id  VARCHAR(100),
    status          VARCHAR(20) NOT NULL,           -- PENDING | SUCCESS | FAILED
    amount_cents    BIGINT NOT NULL,
    idempotency_key VARCHAR(64) UNIQUE NOT NULL,
    created_at      TIMESTAMPTZ DEFAULT NOW()
);
```

### MongoDB — Event Logs & Analytics

```javascript
// Queue events collection
{
  _id: ObjectId,
  saleId: "sale-123",
  userId: "user-abc",
  event: "QUEUED" | "DEQUEUED" | "EXPIRED" | "PURCHASED",
  queuePosition: 42,
  timestamp: ISODate,
  metadata: {
    ipAddress: "...",
    userAgent: "...",
    botScore: 0.95,
    admissionLatencyMs: 23
  }
}

// Sale analytics collection
{
  _id: ObjectId,
  saleId: "sale-123",
  windowStart: ISODate,
  windowEnd: ISODate,
  metrics: {
    totalRequests: 1000000,
    queuedUsers: 1000,
    purchasedUnits: 100,
    avgQueueWaitMs: 12400,
    peakRequestsPerSecond: 890000,
    botRequestsBlocked: 234000
  }
}
```

---

## 7. Redis Data Structures

```
Key Pattern                              Type        TTL       Purpose
───────────────────────────────────────────────────────────────────────────
flash_sale:{saleId}:inventory           STRING      None      Unit counter
flash_sale:{saleId}:queue               ZSET        None      Queue (score=time)
flash_sale:{saleId}:meta                STRING      1 hour    Sale config JSON
flash_sale:{saleId}:active              STRING      None      "1" if sale live

registered:{saleId}:{userId}            STRING      24 hours  Pre-registration
purchased:{saleId}:{userId}             STRING      7 days    Purchase dedup
purchased:{saleId}:{paymentFP}          STRING      7 days    Payment dedup

rate_limit:global:{saleId}              STRING      1 sec     Global token bucket
rate_limit:user:{userId}:{saleId}       STRING      1 min     Per-user limit
rate_limit:ip:{ip}:{saleId}             STRING      10 sec    Per-IP limit

payment_timer:{orderId}                 STRING      300 sec   5-min payment window
idempotency:{idempotencyKey}            STRING      24 hours  Idempotency cache
queue_position:{saleId}:{userId}        STRING      5 sec     Cached position
```

---

## 8. Kafka Event Flow

```
Topic: sale-events
  Partitioned by: saleId (ensures ordering within a sale)
  Retention: 7 days

Events:
  SALE_STARTED       → admission-service enables queue
  USER_QUEUED        → analytics-service logs it
  USER_DEQUEUED      → order-service creates order
  INVENTORY_RESERVED → saga moves to payment step
  PAYMENT_INITIATED  → payment-service processes it
  PAYMENT_SUCCESS    → inventory-service commits, order confirmed
  PAYMENT_FAILED     → inventory-service releases stock, saga compensates
  ORDER_CONFIRMED    → notification-service sends confirmation email/SMS
  ORDER_CANCELLED    → next user in queue notified
  SALE_ENDED         → all services clean up

Topic: queue-position-updates
  Partitioned by: userId (mod N)
  Purpose: notification-service pushes position to WebSocket
  Frequency: every 500ms per active user
  Retention: 1 minute (ephemeral)

Consumer Groups:
  admission-service:   sale-events (SALE_STARTED)
  order-service:       sale-events (USER_DEQUEUED)
  inventory-service:   sale-events (PAYMENT_SUCCESS, PAYMENT_FAILED)
  notification-service: sale-events + queue-position-updates
  analytics-service:   sale-events (all, for logging to MongoDB)
```

**Kafka guarantees used:**

- **Exactly-once semantics** for payment events (idempotency key in message)
- **At-least-once** for queue position updates (tolerate duplicates)
- **Ordering** within a saleId partition (SALE_STARTED before USER_DEQUEUED)

---

## 9. Implementation Code

### 9.1 Flash Sale Admission Service

```java
@RestController
@RequestMapping("/api/v1/flash-sale")
@RequiredArgsConstructor
public class AdmissionController {

    private final AdmissionService admissionService;
    private final RateLimiterService rateLimiterService;

    @PostMapping("/{saleId}/join")
    public ResponseEntity<AdmissionResponse> joinSale(
            @PathVariable String saleId,
            @RequestHeader("Authorization") String token,
            HttpServletRequest request) {

        String userId = jwtService.extractUserId(token);
        String ipAddress = request.getRemoteAddr();

        // 1. Check if sale is active
        if (!admissionService.isSaleActive(saleId)) {
            return ResponseEntity.status(HttpStatus.NOT_FOUND)
                .body(AdmissionResponse.saleNotActive());
        }

        // 2. Check per-user rate limit (1 attempt per user)
        if (!rateLimiterService.allowUser(userId, saleId)) {
            return ResponseEntity.status(HttpStatus.TOO_MANY_REQUESTS)
                .body(AdmissionResponse.alreadyQueued(
                    admissionService.getQueuePosition(saleId, userId)));
        }

        // 3. Check if user already purchased
        if (admissionService.hasPurchased(saleId, userId)) {
            return ResponseEntity.status(HttpStatus.CONFLICT)
                .body(AdmissionResponse.alreadyPurchased());
        }

        // 4. Check if user pre-registered
        if (!admissionService.isRegistered(saleId, userId)) {
            return ResponseEntity.status(HttpStatus.FORBIDDEN)
                .body(AdmissionResponse.notRegistered());
        }

        // 5. Check global rate limit and queue capacity
        AdmissionResult result = admissionService.admit(saleId, userId);

        return switch (result.status()) {
            case QUEUED -> ResponseEntity.ok(
                AdmissionResponse.queued(result.position(), result.estimatedWaitMs()));
            case SOLD_OUT -> ResponseEntity.status(HttpStatus.GONE)
                .body(AdmissionResponse.soldOut());
            case QUEUE_FULL -> ResponseEntity.status(HttpStatus.SERVICE_UNAVAILABLE)
                .body(AdmissionResponse.queueFull());
        };
    }
}

@Service
@RequiredArgsConstructor
public class AdmissionService {

    private final StringRedisTemplate redis;
    private final KafkaTemplate<String, Object> kafkaTemplate;

    private static final int MAX_QUEUE_SIZE = 1000;

    public AdmissionResult admit(String saleId, String userId) {
        String queueKey = "flash_sale:" + saleId + ":queue";
        String inventoryKey = "flash_sale:" + saleId + ":inventory";

        // Check inventory first (fast fail)
        String stock = redis.opsForValue().get(inventoryKey);
        if (stock == null || Integer.parseInt(stock) <= 0) {
            return AdmissionResult.soldOut();
        }

        // Check queue capacity
        Long queueSize = redis.opsForZSet().zCard(queueKey);
        if (queueSize != null && queueSize >= MAX_QUEUE_SIZE) {
            return AdmissionResult.queueFull();
        }

        // Add to sorted set with microsecond timestamp as score (NX = only if not present)
        long score = System.nanoTime() / 1000; // microseconds
        Boolean added = redis.opsForZSet().addIfAbsent(queueKey, userId, score);

        if (Boolean.FALSE.equals(added)) {
            // User already in queue
            Long position = redis.opsForZSet().rank(queueKey, userId);
            return AdmissionResult.queued(position != null ? position + 1 : 0, estimateWait(position));
        }

        Long position = redis.opsForZSet().rank(queueKey, userId);

        // Publish event
        kafkaTemplate.send("sale-events", saleId,
            new UserQueuedEvent(saleId, userId, position, System.currentTimeMillis()));

        return AdmissionResult.queued(position != null ? position + 1 : 1, estimateWait(position));
    }

    private long estimateWait(Long position) {
        if (position == null) return 0;
        return position * 3000L; // rough: 3 seconds per position
    }
}
```

### 9.2 Redis Inventory Lua Script (Atomic Decrement)

```java
@Component
public class InventoryLuaScript {

    // Lua script: atomically check-and-decrement
    // Returns remaining stock, or -1 if sold out
    private static final String DECR_IF_POSITIVE = """
        local current = tonumber(redis.call('GET', KEYS[1]))
        if current == nil then return -2 end
        if current <= 0 then return -1 end
        return redis.call('DECR', KEYS[1])
        """;

    // Restore inventory (on payment failure)
    private static final String INCR_BOUNDED = """
        local current = tonumber(redis.call('GET', KEYS[1]))
        local max = tonumber(ARGV[1])
        if current >= max then return current end
        return redis.call('INCR', KEYS[1])
        """;

    private final RedisScript<Long> decrScript;
    private final RedisScript<Long> incrScript;
    private final StringRedisTemplate redis;

    public InventoryLuaScript(StringRedisTemplate redis) {
        this.redis = redis;
        this.decrScript = RedisScript.of(DECR_IF_POSITIVE, Long.class);
        this.incrScript = RedisScript.of(INCR_BOUNDED, Long.class);
    }

    /**
     * Attempt to reserve 1 unit of inventory.
     * @return remaining stock after decrement, or -1 if sold out
     */
    public long reserveUnit(String saleId) {
        String key = "flash_sale:" + saleId + ":inventory";
        Long result = redis.execute(decrScript, List.of(key));
        return result != null ? result : -2;
    }

    /**
     * Release a previously reserved unit back to inventory.
     */
    public void releaseUnit(String saleId, int maxStock) {
        String key = "flash_sale:" + saleId + ":inventory";
        redis.execute(incrScript, List.of(key), String.valueOf(maxStock));
    }
}
```

### 9.3 Queue Worker Service

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class QueueWorkerService {

    private final StringRedisTemplate redis;
    private final InventoryLuaScript inventory;
    private final OrderService orderService;
    private final KafkaTemplate<String, Object> kafkaTemplate;

    private volatile boolean running = false;

    @Scheduled(fixedDelay = 10) // runs every 10ms
    public void processQueue() {
        if (!running) return;

        // Get active sales
        Set<String> activeSales = getActiveSaleIds();
        for (String saleId : activeSales) {
            processNextInQueue(saleId);
        }
    }

    private void processNextInQueue(String saleId) {
        String queueKey = "flash_sale:" + saleId + ":queue";

        // ZPOPMIN: atomically pop user with lowest score (earliest arrival)
        Set<ZSetOperations.TypedTuple<String>> entries =
            redis.opsForZSet().popMin(queueKey, 1);

        if (entries == null || entries.isEmpty()) return;

        String userId = entries.iterator().next().getValue();
        if (userId == null) return;

        // Attempt to reserve inventory
        long remaining = inventory.reserveUnit(saleId);

        if (remaining < 0) {
            // Sold out — re-enqueue this user? No, the sale is over.
            log.info("Sale {} sold out. User {} missed out.", saleId, userId);
            notifyUserSoldOut(saleId, userId);
            stopSale(saleId);
            return;
        }

        log.info("Sale {} — reserved unit for user {}. {} units remaining.", saleId, userId, remaining);

        // Publish dequeue event → Order Service will create the order
        kafkaTemplate.send("sale-events", saleId,
            new UserDequeuedEvent(saleId, userId, remaining, System.currentTimeMillis()));

        // If last unit sold, end the sale
        if (remaining == 0) {
            stopSale(saleId);
        }
    }

    @KafkaListener(topics = "sale-events", groupId = "queue-worker")
    public void handleSaleEvent(SaleEvent event) {
        if (event instanceof SaleStartedEvent e) {
            seedInventory(e.getSaleId(), e.getTotalUnits());
            running = true;
            log.info("Sale {} started. Inventory seeded: {} units", e.getSaleId(), e.getTotalUnits());
        } else if (event instanceof SaleEndedEvent e) {
            running = false;
        }
    }

    private void seedInventory(String saleId, int units) {
        String inventoryKey = "flash_sale:" + saleId + ":inventory";
        redis.opsForValue().set(inventoryKey, String.valueOf(units));
        redis.opsForValue().set("flash_sale:" + saleId + ":active", "1");
    }
}
```

### 9.4 WebSocket Queue Position Push

```java
// WebSocket configuration
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        registry.enableSimpleBroker("/topic", "/queue");
        registry.setApplicationDestinationPrefixes("/app");
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws/flash-sale")
            .setAllowedOriginPatterns("*")
            .withSockJS();
    }
}

// Position updater — Kafka consumer pushes to WebSocket
@Service
@RequiredArgsConstructor
public class QueuePositionPusher {

    private final SimpMessagingTemplate wsTemplate;
    private final StringRedisTemplate redis;

    @KafkaListener(topics = "queue-position-updates", groupId = "ws-pusher")
    public void handlePositionUpdate(QueuePositionUpdate update) {
        // Push to user's personal channel
        wsTemplate.convertAndSendToUser(
            update.getUserId(),
            "/queue/position",
            new PositionMessage(
                update.getPosition(),
                update.getEstimatedWaitSeconds(),
                update.getTotalQueueSize()
            )
        );
    }

    // Background job: every 500ms, broadcast position updates for all queued users
    @Scheduled(fixedRate = 500)
    public void broadcastPositions() {
        Set<String> activeSales = getActiveSaleIds();
        for (String saleId : activeSales) {
            String queueKey = "flash_sale:" + saleId + ":queue";
            // Get all members with their scores
            Set<ZSetOperations.TypedTuple<String>> members =
                redis.opsForZSet().rangeWithScores(queueKey, 0, -1);

            if (members == null) continue;

            long position = 1;
            for (ZSetOperations.TypedTuple<String> member : members) {
                String userId = member.getValue();
                kafkaTemplate.send("queue-position-updates",
                    userId, // partition key = userId for ordering
                    new QueuePositionUpdate(saleId, userId, position++, members.size()));
            }
        }
    }
}
```

### 9.5 Order Service with Saga Pattern

```java
// Saga orchestrator for flash sale order
@Service
@RequiredArgsConstructor
@Slf4j
public class FlashSaleOrderSaga {

    private final OrderRepository orderRepo;
    private final SagaStateRepository sagaRepo;
    private final KafkaTemplate<String, Object> kafka;
    private final InventoryLuaScript inventory;

    @KafkaListener(topics = "sale-events", groupId = "order-service")
    public void onUserDequeued(UserDequeuedEvent event) {
        // Step 1: Create order in PENDING state
        String idempotencyKey = "order:" + event.getSaleId() + ":" + event.getUserId();

        // Idempotency check
        if (orderRepo.existsByIdempotencyKey(idempotencyKey)) {
            log.warn("Duplicate dequeue event for user {} sale {}", event.getUserId(), event.getSaleId());
            return;
        }

        Order order = Order.builder()
            .saleId(event.getSaleId())
            .userId(event.getUserId())
            .status(OrderStatus.PENDING)
            .priceCents(getSalePrice(event.getSaleId()))
            .idempotencyKey(idempotencyKey)
            .paymentExpiresAt(Instant.now().plusSeconds(300)) // 5 min window
            .build();

        order = orderRepo.save(order);

        // Save saga state
        SagaState saga = SagaState.builder()
            .sagaId(UUID.randomUUID())
            .orderId(order.getId())
            .currentStep("INVENTORY_RESERVED")
            .stepsCompleted(List.of("INVENTORY_RESERVED"))
            .build();
        sagaRepo.save(saga);

        // Step 2: Notify user to proceed to payment
        kafka.send("sale-events", event.getSaleId(),
            new PaymentRequiredEvent(event.getSaleId(), event.getUserId(),
                order.getId(), order.getPriceCents(), order.getPaymentExpiresAt()));

        // Set payment expiry timer in Redis
        redis.opsForValue().set(
            "payment_timer:" + order.getId(), "1",
            Duration.ofSeconds(300)
        );
    }

    @KafkaListener(topics = "sale-events", groupId = "order-service")
    public void onPaymentResult(PaymentResultEvent event) {
        Order order = orderRepo.findById(event.getOrderId()).orElseThrow();
        SagaState saga = sagaRepo.findByOrderId(event.getOrderId()).orElseThrow();

        if (event.isSuccess()) {
            // COMMIT path
            int updated = orderRepo.updateStatusWithVersion(
                event.getOrderId(), OrderStatus.CONFIRMED, OrderStatus.PENDING, order.getVersion()
            );
            if (updated == 0) {
                log.warn("Optimistic lock conflict for order {}", event.getOrderId());
                return; // another thread already handled it
            }

            saga.setCurrentStep("COMPLETED");
            saga.addStep("ORDER_CONFIRMED");
            sagaRepo.save(saga);

            kafka.send("sale-events", order.getSaleId(),
                new OrderConfirmedEvent(order.getSaleId(), order.getUserId(), order.getId()));

        } else {
            // COMPENSATE path
            log.info("Payment failed for order {}. Releasing inventory.", event.getOrderId());

            // Compensating transaction: release inventory
            inventory.releaseUnit(order.getSaleId(), 100); // max=100 units

            orderRepo.updateStatusWithVersion(
                event.getOrderId(), OrderStatus.CANCELLED, OrderStatus.PENDING, order.getVersion()
            );

            saga.setCurrentStep("COMPENSATED");
            saga.addStep("INVENTORY_RELEASED");
            sagaRepo.save(saga);

            kafka.send("sale-events", order.getSaleId(),
                new OrderCancelledEvent(order.getSaleId(), order.getUserId(),
                    order.getId(), "PAYMENT_FAILED"));
        }
    }
}
```

### 9.6 Payment Service

```java
@Service
@RequiredArgsConstructor
public class PaymentService {

    private final PaymentGatewayClient gatewayClient;
    private final PaymentRepository paymentRepo;
    private final StringRedisTemplate redis;
    private final KafkaTemplate<String, Object> kafka;

    @KafkaListener(topics = "sale-events", groupId = "payment-service")
    public void onPaymentRequired(PaymentRequiredEvent event) {
        // Check payment timer hasn't expired
        String timerKey = "payment_timer:" + event.getOrderId();
        if (!Boolean.TRUE.equals(redis.hasKey(timerKey))) {
            log.warn("Payment window expired for order {}", event.getOrderId());
            kafka.send("sale-events", event.getSaleId(),
                new PaymentResultEvent(event.getOrderId(), false, "WINDOW_EXPIRED"));
            return;
        }

        // Idempotency: don't double-charge
        String idempotencyKey = "payment:" + event.getOrderId();
        Payment existingPayment = paymentRepo.findByIdempotencyKey(idempotencyKey).orElse(null);

        if (existingPayment != null) {
            // Return cached result
            boolean success = existingPayment.getStatus() == PaymentStatus.SUCCESS;
            kafka.send("sale-events", event.getSaleId(),
                new PaymentResultEvent(event.getOrderId(), success, existingPayment.getGatewayTxnId()));
            return;
        }

        // Process payment (user's saved payment method)
        try {
            PaymentResult result = gatewayClient.charge(ChargeRequest.builder()
                .userId(event.getUserId())
                .amountCents(event.getAmountCents())
                .idempotencyKey(idempotencyKey)
                .description("Flash Sale Order " + event.getOrderId())
                .build());

            Payment payment = Payment.builder()
                .orderId(event.getOrderId())
                .gateway(result.getGateway())
                .gatewayTxnId(result.getTransactionId())
                .status(result.isSuccess() ? PaymentStatus.SUCCESS : PaymentStatus.FAILED)
                .amountCents(event.getAmountCents())
                .idempotencyKey(idempotencyKey)
                .build();

            paymentRepo.save(payment);

            // Clear payment timer
            redis.delete(timerKey);

            kafka.send("sale-events", event.getSaleId(),
                new PaymentResultEvent(event.getOrderId(), result.isSuccess(), result.getTransactionId()));

        } catch (PaymentGatewayException e) {
            log.error("Gateway error for order {}: {}", event.getOrderId(), e.getMessage());
            kafka.send("sale-events", event.getSaleId(),
                new PaymentResultEvent(event.getOrderId(), false, "GATEWAY_ERROR"));
        }
    }
}
```

---

## 10. Failure Scenarios & Mitigations

### Scenario 1: Redis Goes Down Mid-Sale

```
Problem: Inventory key lost → can't decrement → sale halts

Mitigation:
  1. Redis Sentinel / Redis Cluster (3+ nodes, auto-failover <30s)
  2. Before sale: persist inventory to PostgreSQL as backup
  3. On Redis reconnect: re-seed inventory from PostgreSQL
     (atomic: only re-seed if Redis key doesn't exist)
  4. During outage: circuit breaker → "sale paused" message to users

Recovery:
  - Read last confirmed order count from PostgreSQL
  - Re-seed Redis: SET flash_sale:X:inventory (100 - confirmed_count)
  - Re-activate queue worker
```

### Scenario 2: User Pays But Order Not Confirmed (Kafka Lag)

```
Problem: Payment succeeds at gateway, but Kafka consumer is behind
         User sees "Processing" for too long

Mitigation:
  1. Payment service stores result immediately in PostgreSQL
  2. User can poll GET /orders/{orderId}/status
  3. Order service has idempotency: duplicate Kafka messages are safe
  4. Kafka consumer lag alert: PagerDuty if lag > 1000 messages
  5. Timeout: if order not confirmed in 60s, user sees error + refund triggered
```

### Scenario 3: Queue Worker Crashes Mid-Pop

```
Problem: ZPOPMIN removes user from queue, worker crashes before creating order
         User lost from queue, inventory not reserved

Mitigation:
  1. Two-phase queue pop using Redis transactions:
     - ZADD to "processing" set (with TTL)
     - ZREM from main queue
  2. Background monitor: if "processing" entry older than 10s with no order,
     re-insert into main queue

  // Safer pop pattern
  String processingKey = "flash_sale:" + saleId + ":processing";
  redis.multi();
    redis.opsForZSet().add(processingKey, userId, System.currentTimeMillis());
    redis.expire(processingKey, Duration.ofSeconds(10));
  redis.exec();
```

### Scenario 4: 1M Users Flood at Exactly T=0 (Cache Stampede)

```
Problem: At 12:00:00.000, 1M users all send requests simultaneously
         API Gateway overwhelmed, Redis flooded with connections

Mitigation:
  1. Virtual queue with jitter: each user given a random 0-500ms delay
     Server-side: process admissions in 100ms batches
  2. Pre-admission phase (T-5min to T+0): accept queue registrations
     Process in order at T+0 (smoothed burst)
  3. Connection pooling: Redis connection pool sized for burst
     spring.data.redis.lettuce.pool.max-active=200
  4. API Gateway autoscaling pre-triggered at T-10min
```

### Scenario 5: Payment Timer Expiry Race Condition

```
Problem: User submits payment at T=299s (1 second before expiry)
         Timer expires at T=300s while payment is in-flight

Mitigation:
  1. Payment timer check in payment service uses a 30-second grace window
     Actual: TTL=330s, but display to user as 5 minutes
  2. If payment completes after expiry but before 330s:
     Accept the payment (grace period)
  3. If payment completes after 330s:
     Attempt refund via gateway, release inventory

  Key insight: the timer is for UX (urgency) — the real guard is
  the Redis inventory decrement, which already happened.
```

---

## 11. Scaling Strategy

### Horizontal Scaling by Layer

```
Layer                   Scale Unit        Target Capacity
──────────────────────────────────────────────────────────────
CDN (Cloudflare)        Edge nodes        Unlimited (global)
API Gateway             Pods (K8s)        50 pods × 20K req/s = 1M req/s
Admission Service       Pods              20 pods (stateless)
Queue Service           Pods              5 pods (Redis is the state)
Order Service           Pods              10 pods (Kafka consumers)
Payment Service         Pods              10 pods (I/O bound)
Notification Service    Pods              20 pods (WebSocket sticky)
Redis                   Cluster (6 nodes) 500K ops/s
Kafka                   Brokers           6 brokers, 12 partitions
PostgreSQL              1 primary + 2 RR  10K writes/s
MongoDB                 Replica set (3)   50K writes/s
```

### Redis Cluster Sharding

```
flash_sale:{saleId}:inventory  → slot based on saleId hash
flash_sale:{saleId}:queue      → same slot (same hash tag {saleId})

Redis Cluster hash tags ensure both keys land on same node:
  {flash_sale:123}:inventory
  {flash_sale:123}:queue
  → Both hash to the same slot → both on same node → Lua scripts work
```

### Kafka Partitioning

```
Topic: sale-events
  Partitions: 12
  Partition key: saleId
  All events for a sale → same partition → guaranteed ordering

Topic: queue-position-updates
  Partitions: 50
  Partition key: userId (hash mod 50)
  High volume, no ordering needed within partition
```

---

## 12. Monitoring & Observability

### Key Metrics (Prometheus + Grafana)

```
flash_sale_queue_depth{saleId}       → current queue length
flash_sale_inventory_remaining{saleId} → units left (from Redis)
flash_sale_requests_per_second       → rate at API gateway
flash_sale_admission_latency_p99     → <50ms target
flash_sale_orders_confirmed_total    → should reach exactly 100
flash_sale_payment_failures_total    → triggers compensation
queue_worker_process_time_ms         → time to process each user
redis_connected_clients              → connection pool health
kafka_consumer_lag{group, topic}     → <1000 messages
```

### Alerts

```
CRITICAL:
  flash_sale_inventory_remaining < 0   → Oversell detected! Page on-call
  redis_down                           → Sale halted, failover
  kafka_consumer_lag > 5000            → Order processing delayed

WARNING:
  flash_sale_queue_depth > 800         → Queue near capacity
  payment_failure_rate > 20%           → Gateway issue
  admission_latency_p99 > 100ms        → Scale API gateway pods
```

### Distributed Tracing (OpenTelemetry)

```
Trace spans per purchase:
  api_gateway.admit          5ms
  admission_service.check    10ms
  redis.zadd                 1ms
  ──────────────────────────────
  queue_worker.pop           1ms
  redis.decr_inventory       1ms
  kafka.publish              5ms
  ──────────────────────────────
  order_service.create       20ms
  postgresql.insert          15ms
  ──────────────────────────────
  payment_service.charge     800ms (external gateway)
  kafka.publish_result       5ms
  ──────────────────────────────
  order_service.confirm      10ms
  notification.send          50ms
  ──────────────────────────────
  Total (happy path):        ~930ms
```

---

## 13. Summary Cheat Sheet

### The 6 Core Answers at a Glance

| Question | Answer | Technology |
|---|---|---|
| Prevent overselling | Atomic Lua DECR in Redis; DB is audit log | Redis Lua script |
| Handle 1M users | 5-layer load shedding; queue only 100-1000 users | CDN + Gateway + Redis |
| Locking strategy | Redis atomic (no locks) for inventory; Optimistic lock for order state | Redis + PostgreSQL version column |
| Waiting queue | Redis ZSET with microsecond score; ZPOPMIN to dequeue fairly | Redis Sorted Set |
| Fairness / anti-bot | Pre-registration + CAPTCHA + behavioral scoring + 1-per-user Redis guard | Redis SETNX + MongoDB scoring |
| Caching strategy | Redis = inventory source of truth; L1+L2 for sale metadata; CDN for static | Caffeine + Redis + CDN |

### Golden Rules for Flash Sales

```
1. NEVER use a SQL row lock as the primary inventory guard under high concurrency
2. Redis atomic operations (Lua scripts) are the only safe inventory counter
3. Shed load at the edge — only ~100 requests should reach business logic
4. Pre-warm EVERYTHING: CDN, Redis, L1 cache, DB connections — before the sale starts
5. The queue is in Redis; PostgreSQL is only written when payment succeeds
6. Idempotency keys everywhere — payment gateway, order creation, Kafka consumers
7. Design for compensating transactions — payment failures WILL happen
8. Set a payment timer (5 min) — holding inventory indefinitely causes starvation
9. Fairness = timestamp ordering in ZSET — not application-level random selection
10. Test with 10x expected load BEFORE the sale; never discover limits during one
```

### Technology Role Summary

```
Redis:        ★ Inventory counter (source of truth)
              ★ Queue (ZSET with time-based scoring)
              ★ Rate limiting (token bucket per user/IP)
              ★ Idempotency key cache
              ★ Payment timer (TTL key)

Kafka:        ★ Decouple admission → order → payment → notification
              ★ Guaranteed ordering within a sale partition
              ★ Replay events for recovery
              ★ Buffer burst traffic to downstream services

PostgreSQL:   ★ Durable order records (ACID)
              ★ Payment records (financial audit)
              ★ Saga state (distributed transaction tracking)
              ★ Flash sale configuration

MongoDB:      ★ Queue event log (high-write analytics)
              ★ Behavioral scoring data (bot detection)
              ★ Sale performance metrics
              ★ Audit trail (compliance)

Spring Boot:  ★ Kafka consumers (order, payment, notification)
              ★ WebSocket server (queue position push)
              ★ Scheduled jobs (queue worker, position broadcaster)
              ★ Redis Lua script execution
```
