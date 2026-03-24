# Shopping Cart System — Deep Dive Design

> **Scenario:** Distributed shopping cart for 50,000 concurrent users.
> Anonymous (session) carts + persistent (user) carts + cart merge on login.
> Live price display · 30-day inactivity expiry · 100,000 simultaneous adds during flash sale.

**Tech Stack:** Java 17 · Spring Boot 3 · PostgreSQL · MongoDB · Redis · Kafka · Spring Session

---

## Table of Contents

1. [Requirements & Constraints](#1-requirements--constraints)
2. [Capacity Estimation](#2-capacity-estimation)
3. [High-Level Architecture](#3-high-level-architecture)
4. [Core Design Questions Answered](#4-core-design-questions-answered)
   - 4.1 Where do you store cart data?
   - 4.2 How do you handle cart merging when user logs in?
   - 4.3 How do you handle price changes for items in cart?
   - 4.4 How do you prevent race conditions during flash sales?
   - 4.5 How do you implement cart expiration efficiently?
5. [Data Models](#5-data-models)
6. [Redis Data Structures](#6-redis-data-structures)
7. [Cart State Machine](#7-cart-state-machine)
8. [Kafka Event Flow](#8-kafka-event-flow)
9. [Implementation Code](#9-implementation-code)
   - 9.1 Cart Service — Core Operations
   - 9.2 Anonymous Cart (Session-Based)
   - 9.3 Cart Merge Strategy
   - 9.4 Price Snapshot & Live Price Enrichment
   - 9.5 Flash Sale Race-Condition Guard (Lua Script)
   - 9.6 Cart Expiry — TTL + Sliding Window
   - 9.7 Cart Persistence Worker (Redis → MongoDB)
10. [Failure Scenarios & Mitigations](#10-failure-scenarios--mitigations)
11. [Scaling Strategy](#11-scaling-strategy)
12. [Monitoring & Observability](#12-monitoring--observability)
13. [Summary Cheat Sheet](#13-summary-cheat-sheet)

---

## 1. Requirements & Constraints

### Functional Requirements

- **Anonymous cart** — guest users can add/remove/update items via session cookie (no login required)
- **Persistent cart** — logged-in users have cart synced across all devices and sessions
- **Cart merge** — when a guest logs in, their anonymous cart merges with their existing user cart
- **Live pricing** — every cart render shows the *current* product price (not the price at add-time), but preserves the price snapshot for order locking
- **Cart expiry** — carts expire after **30 days of inactivity** (sliding window — activity resets the clock)
- **Flash sale support** — up to 100,000 users can add the same flash-sale item simultaneously without overselling or data corruption
- **Item quantity limits** — configurable per-product max quantity per cart (e.g., max 3 during flash sales)
- **Coupon / promo codes** — can be applied to cart (out of scope for implementation, but schema accommodates)

### Non-Functional Requirements

- **Concurrency:** 50,000 concurrent users; 100,000 simultaneous add-to-cart during flash sales
- **Latency:** p99 < 30ms for add-to-cart; p99 < 50ms for full cart read
- **Availability:** 99.99% — cart must be readable even if persistence layer is temporarily degraded
- **Durability:** Logged-in user cart must survive service restarts and Redis eviction
- **Idempotency:** Double-clicking "Add to Cart" must not add 2 items
- **Consistency model:** Strong consistency for quantity (no negative stock holds); eventual for price display

### Out of Scope

- Payment processing, order creation (see Question 3)
- Inventory reservation (done at checkout, not at add-to-cart)
- Wishlist (separate service with similar but distinct semantics)

---

## 2. Capacity Estimation

```
Concurrent users:        50,000
Flash sale peak:         100,000 add-to-cart/sec (burst, seconds long)
Average cart ops:        5 ops/user/session = 250,000 ops/sec (normal peak)
Avg items per cart:      4.2
Avg cart payload size:   ~2 KB (4-5 items with thumbnails/names stripped to IDs)

Redis cart storage:
  50,000 active carts × 2 KB = 100 MB (trivial)
  30-day retention: 50,000 users × 30 days × 2 KB = 3 GB
  → Redis can hold all active carts in memory easily

MongoDB cart persistence:
  50,000 users × 2 KB avg cart = 100 MB
  With 1 year history: 1M users × 2 KB = 2 GB

PostgreSQL cart_events (for auditing):
  50,000 users × 10 events/day × 365 days = 182M rows/year
  Each row ~200 bytes = ~36 GB/year → partition by month

Throughput:
  Normal:        250,000 cart ops/sec → Redis handles easily (500K ops/sec/node)
  Flash sale:    100,000 simultaneous add ops → need Lua scripts + rate limiting
  Peak writes:   ~100,000 HSET/sec to Redis → well within capacity

Read/Write ratio:   ~8:1 (browsing cart >> adding items)
```

---

## 3. High-Level Architecture

```
                   ┌────────────────────────────────────────────────────────┐
                   │                   CLIENT LAYER                         │
                   │    Browser  /  Mobile App  /  Kiosk Terminal           │
                   │    - Session cookie (anonymous: sessionId)             │
                   │    - JWT Bearer token (logged in: userId)              │
                   └───────────────────────┬────────────────────────────────┘
                                           │
                   ┌───────────────────────▼────────────────────────────────┐
                   │             API GATEWAY (Kong / Spring Cloud)          │
                   │   JWT validation · Cookie parsing · Rate limiting      │
                   │   Routes by auth state → cart-service                  │
                   └───────────────────────┬────────────────────────────────┘
                                           │
                   ┌───────────────────────▼────────────────────────────────┐
                   │                  CART SERVICE                          │
                   │   (Stateless Spring Boot, horizontally scaled)         │
                   │                                                         │
                   │   ┌─────────────────┐   ┌────────────────────────┐    │
                   │   │  Cart Command   │   │    Cart Query          │    │
                   │   │  - Add item     │   │    - Get cart          │    │
                   │   │  - Remove item  │   │    - Enrich prices     │    │
                   │   │  - Update qty   │   │    - Calculate totals  │    │
                   │   │  - Apply coupon │   │    - Validate stock    │    │
                   │   │  - Merge cart   │   └──────────┬─────────────┘    │
                   │   └────────┬────────┘              │                  │
                   └────────────┼───────────────────────┼──────────────────┘
                                │                       │
          ┌─────────────────────▼───┐        ┌──────────▼───────────────┐
          │        REDIS            │        │    PRICE SERVICE         │
          │  ┌──────────────────┐   │        │    (read-only lookup)    │
          │  │ Anonymous Carts  │   │        │    GET /prices/{skuId}   │
          │  │ Key: cart:s:{sid}│   │        └──────────────────────────┘
          │  ├──────────────────┤   │
          │  │  User Carts      │   │
          │  │ Key: cart:u:{uid}│   │        ┌──────────────────────────┐
          │  ├──────────────────┤   │        │  PRODUCT SERVICE         │
          │  │  Flash Sale      │   │        │  (stock/availability)    │
          │  │  Rate Limiters   │   │        └──────────────────────────┘
          │  ├──────────────────┤   │
          │  │  Idempotency     │   │
          │  │  Keys            │   │
          │  └──────────────────┘   │
          └─────────────────────────┘
                       │
            ┌──────────▼─────────────────────────────────────────────────┐
            │                       KAFKA                                │
            │   cart-events  ·  cart-merged  ·  cart-expired             │
            └──────┬────────────────────┬────────────────────────────────┘
                   │                    │
       ┌───────────▼─────┐    ┌─────────▼──────────────────────────────┐
       │  CART PERSISTER │    │  ANALYTICS / RECOMMENDATION SERVICE   │
       │  (Kafka consumer│    │  (track add-to-cart, abandonment)      │
       │   Redis → MongoDB│   └────────────────────────────────────────┘
       │   Redis → PgSQL) │
       └─────────────────┘

Data Stores:
  ┌─────────────────┐  ┌─────────────────────┐  ┌──────────────────┐
  │   REDIS         │  │   MONGODB           │  │   POSTGRESQL     │
  │  Primary cart   │  │  Durable cart store │  │  Cart events     │
  │  store (speed)  │  │  (user carts)       │  │  Audit log       │
  │  TTL management │  │  Merge history      │  │  Coupon tracking │
  │  Flash sale     │  │  Cart snapshots     │  │  Analytics       │
  │  rate limits    │  │  Abandoned carts    │  │                  │
  └─────────────────┘  └─────────────────────┘  └──────────────────┘
```

---

## 4. Core Design Questions Answered

---

### 4.1 Where Do You Store Cart Data?

**Answer: Redis as the primary store (speed + TTL), MongoDB as the durable persistence layer for logged-in users.**

The two types of cart have very different lifecycle requirements:

```
Anonymous Cart (Guest User):
  Identity:   sessionId (UUID in cookie, set by API gateway)
  Lifetime:   Short — typically browser session to a few hours; max 7 days
  Storage:    Redis ONLY (no DB write needed — guest data is ephemeral)
  Key:        cart:session:{sessionId}
  TTL:        7 days (absolute) — short enough to not waste Redis memory
  On login:   Merge into user cart, then DELETE the session cart

Logged-In User Cart:
  Identity:   userId (from JWT)
  Lifetime:   Long — 30 days from last activity (sliding window)
  Storage:    Redis (primary, fast) + MongoDB (durable backup)
  Redis key:  cart:user:{userId}
  Redis TTL:  30 days, reset on every access (sliding window)
  MongoDB:    Written asynchronously via Kafka consumer
              Acts as recovery source if Redis evicts the cart
  On every write: Update Redis immediately, publish Kafka event for MongoDB sync

Why not PostgreSQL as primary cart store?
  10,000+ cart reads/sec × avg 50ms SQL query = system melts
  Cart is read-heavy, schema-flexible (variable item attributes),
  and tolerates brief eventual consistency → Redis + MongoDB is a better fit
  PostgreSQL used only for immutable audit log (cart_events table)

Why not MongoDB as primary cart store?
  Even MongoDB at 10ms/query is too slow when Redis can do <1ms
  MongoDB is the durable fallback, not the hot path
```

**Storage decision matrix:**

| Cart Type | Hot Path | Durable Store | TTL Owner | Recovery |
|---|---|---|---|---|
| Anonymous / Guest | Redis Hash | None (ephemeral) | Redis TTL | None needed |
| Logged-in user | Redis Hash | MongoDB document | Redis TTL (sliding) | MongoDB → Redis rebuild |
| Checked-out cart | None | PostgreSQL orders | N/A | Order service owns it |
| Abandoned cart | Redis (cold) | MongoDB snapshot | Redis TTL expires | Remarketing from MongoDB |

---

### 4.2 How Do You Handle Cart Merging When User Logs In?

**This is the trickiest part of cart design.** When a guest logs in, they may have:
- Items only in guest cart
- Items only in their existing user cart (added from another device previously)
- The **same item in both carts** (the conflict case)

**Three merge strategies — choose based on business rules:**

```
Strategy A — Union (Add Together)
  Guest cart:  [iPhone×1, AirPods×1]
  User cart:   [iPhone×2, MacBook×1]
  Merged:      [iPhone×3, AirPods×1, MacBook×1]
  → Simple: sum quantities for same SKU
  → Risk: user accidentally ends up with quantity they didn't intend
  → Best for: add-to-cart dominant workflows (e-commerce)

Strategy B — Guest Cart Wins (Replace)
  Guest cart:  [iPhone×1, AirPods×1]
  User cart:   [iPhone×2, MacBook×1]
  Merged:      [iPhone×1, AirPods×1]
  → Guest cart entirely replaces user cart
  → Safest for user intent (they were actively shopping as guest)
  → Risk: lose items the user carefully added on another device

Strategy C — User Cart Wins (Keep Server State)
  Guest cart:  [iPhone×1, AirPods×1]
  User cart:   [iPhone×2, MacBook×1]
  Merged:      [iPhone×2, MacBook×1]
  → Discard guest cart entirely
  → Risk: user loses items they just added

Strategy D — Smart Merge (Recommended for production)
  For each SKU:
    If SKU only in guest cart:   ADD to user cart
    If SKU only in user cart:    KEEP in user cart
    If SKU in both carts:        TAKE MAX(guest_qty, user_qty)
                                 (assumes higher qty is the user's intent)
  Guest cart:  [iPhone×1, AirPods×1]
  User cart:   [iPhone×2, MacBook×1]
  Merged:      [iPhone×2, AirPods×1, MacBook×1]  ← max(1,2)=2 for iPhone
  → Preserves all items the user has shown interest in
  → Never decreases quantity (safe default)
  → Show user "Your cart was updated" banner with diff
```

**Merge execution — atomic in Redis with Lua:**

```
Merge algorithm (runs in cart-service on login event):

1. Read guest cart: HGETALL cart:session:{sessionId}
2. Read user cart:  HGETALL cart:user:{userId}
3. Compute merged cart using Strategy D (in application memory)
4. Atomic write: Lua script
   - DEL cart:session:{sessionId}       ← remove guest cart atomically
   - HSET cart:user:{userId} ...merged  ← write merged result
   - EXPIRE cart:user:{userId} 2592000  ← reset 30-day TTL
5. Publish cart-merged Kafka event      ← for MongoDB sync + analytics
6. Return merged cart to client with diff

Atomicity: If step 4 Lua script fails, both carts remain unchanged
           Next login attempt will retry the merge safely (idempotent)
```

**Merge is idempotent** — if the user clicks login twice or the network retries:

```java
// Idempotency: track merge completion in Redis
String mergeKey = "cart:merge:" + userId + ":" + sessionId;
Boolean alreadyMerged = redis.opsForValue().setIfAbsent(mergeKey, "done", Duration.ofHours(1));
if (Boolean.FALSE.equals(alreadyMerged)) {
    // Already merged — return current user cart without re-merging
    return getCart(userId);
}
// Proceed with merge...
```

---

### 4.3 How Do You Handle Price Changes for Items in Cart?

**The dual-price model: store the price at add-time (snapshot) but display the current price (live).**

```
Two prices per cart item:
  snapshotPriceCents:  The price when the item was added to cart
  currentPriceCents:   The live price fetched from price-service at read time

Why two prices?
  snapshotPrice: Used for order creation (lock-in at checkout)
                 Provides the "was $X, now $Y" UI experience
                 Required for promotions ("added when on sale")

  currentPrice:  What the user will actually pay today
                 Fetched fresh on every cart read
                 May be higher or lower than snapshot

Cart total = SUM(currentPrice × qty) for all items
```

**Price enrichment at cart read time:**

```
GET /cart (user reads cart):
  1. Fetch cart from Redis: {skuId → {qty, snapshotPrice, addedAt}}
  2. For each SKU, fetch currentPrice from price-service (Redis Hash lookup)
  3. Assemble enriched response:
     {
       items: [
         {
           sku: "APPLE-IPHONE-15",
           qty: 1,
           snapshotPriceCents: 99900,   // when added
           currentPriceCents:  89900,   // right now (10% off!)
           priceDelta:         -10000,  // current is cheaper ✅
           priceDeltaPct:     -10.0,
           priceAlert:        "PRICE_DROPPED"
         },
         {
           sku: "SAMSUNG-EARBUDS",
           qty: 2,
           snapshotPriceCents: 4900,
           currentPriceCents:  5900,    // price went up ⚠️
           priceDelta:         +1000,
           priceDeltaPct:     +20.4,
           priceAlert:        "PRICE_INCREASED"
         }
       ],
       subtotalCents: 101700,           // at current prices
       savedCents:    -9000             // vs snapshot prices (net)
     }
```

**Price change alerts in UI:**

```
Cases and UX handling:
  currentPrice < snapshotPrice → "🎉 Price dropped! Save $X"  → encourage checkout
  currentPrice > snapshotPrice → "⚠️ Price increased by $X"   → create urgency
  currentPrice == snapshotPrice → no alert shown              → clean UI

At checkout confirmation:
  System re-fetches live prices one final time
  If currentPrice differs from displayed price by >1%:
    Show "Price changed since you last viewed cart"
    User must re-confirm before placing order
  This prevents stale-price checkout disputes
```

**Flash sale price handling:**

```
Flash sale starts: product price drops from $999 to $499
  - Price-service pushes PriceChangedEvent to Kafka
  - All cart instances' next cart reads pick up new price from Redis
  - No need to update any cart records
  - TTL on price cache = 60s max → stale window is bounded

Flash sale ends: price returns to $999
  - Same flow; users who didn't checkout now see $999
  - Snapshot price ($499) is preserved in their cart record
  - At checkout: show "Flash sale ended, current price is $999"
  - Force user to acknowledge before proceeding
```

---

### 4.4 How Do You Prevent Race Conditions During Flash Sales?

**The problem:** 100,000 users all click "Add to Cart" for an item with qty_limit=5 per cart. Without protection: concurrent reads of the same cart state all see qty=0, then all write qty=1, netting qty=1 instead of correctly rejecting some requests.

**Worse: 100,000 users adding to carts for a product with only 100 units available.** Cart additions don't reserve inventory (that happens at checkout), but we must enforce *per-cart quantity limits* without race conditions.

**Solution 1: Redis Lua Script for Atomic Quantity Check-and-Set**

```lua
-- Atomically: check quantity limit, then increment if allowed
-- KEYS[1] = cart key  (cart:user:{userId})
-- KEYS[2] = idempotency key
-- ARGV[1] = skuId
-- ARGV[2] = deltaQty (amount to add)
-- ARGV[3] = maxQty (per-cart limit)
-- ARGV[4] = idempotencyTTL (seconds)
-- ARGV[5] = snapshotPrice

-- Idempotency check first
local alreadyProcessed = redis.call('SET', KEYS[2], '1', 'NX', 'EX', ARGV[4])
if alreadyProcessed == false then
    return {-2, 0}  -- duplicate request, already processed
end

-- Get current quantity for this SKU in this cart
local currentQty = tonumber(redis.call('HGET', KEYS[1], ARGV[1] .. ':qty') or '0')
local newQty = currentQty + tonumber(ARGV[2])

-- Enforce per-cart quantity limit
if newQty > tonumber(ARGV[3]) then
    return {-1, currentQty}  -- -1 = limit exceeded
end

-- Safe to update: set qty and snapshot price atomically
redis.call('HSET', KEYS[1], ARGV[1] .. ':qty', newQty)
redis.call('HSET', KEYS[1], ARGV[1] .. ':snapshotPrice', ARGV[5])
redis.call('HSET', KEYS[1], ARGV[1] .. ':addedAt', redis.call('TIME')[1])

-- Refresh cart TTL on any write
redis.call('EXPIRE', KEYS[1], 2592000)  -- 30 days

return {newQty, currentQty}  -- {newQty, prevQty}
```

**Why Lua?** Redis executes Lua scripts atomically — no other command can run between the HGET and HSET. This is the only way to guarantee no race condition without distributed locks.

**Solution 2: Distributed Lock for Cross-Cart Flash Sale Limits**

When a flash sale has a *global* limit (e.g., "max 1 per customer account"), you need a distributed lock:

```java
// RedLock pattern via Redisson for cross-user flash sale limits
public AddToCartResult addToCartWithGlobalLimit(String userId, String skuId, int qty) {
    String lockKey = "flash:lock:" + skuId;
    RLock lock = redissonClient.getLock(lockKey);

    try {
        // Try to acquire lock, wait max 100ms, hold max 500ms
        if (!lock.tryLock(100, 500, TimeUnit.MILLISECONDS)) {
            return AddToCartResult.rateLimited("Too many requests, please retry");
        }

        // Check global cart count for this flash item (across all users)
        Long totalCarted = redis.opsForValue().increment("flash:carted:" + skuId, qty);
        long flashLimit = getFlashSaleLimit(skuId);  // e.g., 100 units

        if (totalCarted > flashLimit) {
            // Undo the increment
            redis.opsForValue().decrement("flash:carted:" + skuId, qty);
            return AddToCartResult.soldOut("Item is sold out");
        }

        // Proceed with cart update (using Lua script for atomicity)
        return executeCartAddLua(userId, skuId, qty);

    } finally {
        if (lock.isHeldByCurrentThread()) lock.unlock();
    }
}
```

**Solution 3: Token Bucket Rate Limiting Per SKU During Flash Sales**

For the traffic spike itself (100,000 req/sec for one item), rate-limit before business logic:

```java
// Sliding window rate limiter in Redis for flash-sale SKU
public boolean allowFlashSaleAdd(String skuId, String userId) {
    String windowKey = "flash:rate:" + skuId + ":" + (System.currentTimeMillis() / 1000);
    long count = redis.opsForValue().increment(windowKey);
    redis.expire(windowKey, Duration.ofSeconds(2));  // 2-second window

    long limit = 5000;  // allow max 5,000 cart-adds per second for this SKU
    if (count > limit) {
        return false;  // politely reject with "Try again in a moment"
    }

    // Also enforce 1 add per user per flash SKU
    String userKey = "flash:user:" + userId + ":" + skuId;
    Boolean firstTime = redis.opsForValue().setIfAbsent(userKey, "1", Duration.ofHours(1));
    return Boolean.TRUE.equals(firstTime);
}
```

**Race condition summary table:**

| Scenario | Risk | Solution |
|---|---|---|
| Double-click "Add to Cart" | Qty incremented twice | Idempotency key (SETNX in Lua) |
| Concurrent adds to same cart | Qty count corrupted | Lua atomic HGET+HSET |
| Flash sale global limit | More than N items carted | Distributed lock (Redisson) |
| 100K simultaneous requests | Server overload | Token bucket rate limiter per SKU |
| Cart merge race (login) | Both carts corrupted | Merge idempotency key + Lua atomic replace |

---

### 4.5 How Do You Implement Cart Expiration Efficiently?

**Requirement: cart expires 30 days after *last activity* (sliding window, not fixed TTL from creation).**

**Mechanism: Redis TTL + sliding window refresh + background sweep for MongoDB cleanup.**

```
Redis TTL (primary mechanism):
  EXPIRE cart:user:{userId} 2592000  ← 2,592,000 seconds = 30 days
  Called on EVERY cart write AND every cart read
  → Each time user views or modifies cart: clock resets to +30 days
  → Costs one EXPIRE command per request (negligible)

When Redis TTL fires:
  Key is deleted from Redis automatically
  User's cart is gone from the fast path
  MongoDB copy still exists (for abandoned cart analytics/remarketing)

MongoDB cleanup:
  TTL index on MongoDB: db.carts.createIndex({lastActivityAt: 1}, {expireAfterSeconds: 2592000})
  MongoDB automatically deletes expired carts matching the TTL index
  → No separate cleanup job needed

For cart reads after Redis eviction:
  Redis miss → check MongoDB → if found AND not expired → rebuild Redis key
  If MongoDB TTL has also fired → cart is truly expired → return empty cart
```

**Sliding window correctness:**

```
Day 0:    User adds item → EXPIRE cart:user:X 2592000  (expires Day 30)
Day 15:   User views cart → EXPIRE cart:user:X 2592000  (now expires Day 45)
Day 45:   User views cart → EXPIRE cart:user:X 2592000  (now expires Day 75)
Day 105:  User does nothing for 30 days → TTL fires → cart deleted ✅

Compare with fixed TTL (wrong):
  Day 0:   EXPIRE 30 days
  Day 29:  User views cart → cart deleted next day (Day 30) ← confusing!
  Sliding window is correct for "30 days of inactivity" semantics
```

**Background job for abandoned cart analytics (every hour):**

```java
// Find carts that haven't been accessed for X days (but not yet expired)
// Used for abandoned cart emails, not for deletion
@Scheduled(cron = "0 0 * * * *")  // every hour
public void flagAbandonedCarts() {
    Instant abandonedThreshold = Instant.now().minus(Duration.ofDays(1));
    Instant expiredThreshold   = Instant.now().minus(Duration.ofDays(30));

    // MongoDB query: active carts not touched in 1+ day, not yet 30-day expired
    List<Cart> abandonedCarts = mongoCartRepo.findByLastActivityAtBetweenAndStatus(
        expiredThreshold, abandonedThreshold, CartStatus.ACTIVE
    );

    for (Cart cart : abandonedCarts) {
        if (cart.getStatus() != CartStatus.ABANDONED) {
            cart.setStatus(CartStatus.ABANDONED);
            mongoCartRepo.save(cart);
            kafkaTemplate.send("cart-events",
                new CartAbandonedEvent(cart.getUserId(), cart.getItems(), cart.getSubtotalCents()));
        }
    }
}
```

---

## 5. Data Models

### Redis Cart Structure (Hash per Cart)

```
Key: cart:user:{userId}          Key: cart:session:{sessionId}
Type: Hash                       Type: Hash
TTL:  30 days (sliding)          TTL:  7 days (fixed)

Fields:
  {skuId}:qty             → "2"           (integer as string)
  {skuId}:snapshotPrice   → "99900"       (cents, at add-time)
  {skuId}:addedAt         → "1699800000"  (unix timestamp)
  {skuId}:metadata        → JSON string   (name, imageUrl, etc. — denormalized for speed)
  __version               → "14"          (optimistic concurrency version)
  __createdAt             → "1699700000"  (cart creation timestamp)
  __lastActivityAt        → "1699800000"  (last write, for analytics)
  __couponCode            → "SAVE10"      (optional)
  __storeId               → "NYC-001"     (store context for pricing)
```

### MongoDB Cart Document (Durable Store)

```javascript
// Collection: carts
{
  _id: ObjectId,
  cartId: "cart-uuid-v4",
  userId: "user-uuid",           // null for anonymous (sessionId instead)
  sessionId: null,               // set for anonymous carts before login
  status: "ACTIVE",              // ACTIVE | ABANDONED | CHECKED_OUT | EXPIRED
  storeId: "NYC-001",
  version: 14,                   // monotonic counter, matches Redis __version
  lastActivityAt: ISODate("2026-03-20T14:30:00Z"),
  createdAt: ISODate("2026-02-18T09:15:00Z"),
  expiresAt: ISODate("2026-04-19T14:30:00Z"),  // computed: lastActivity + 30d

  items: [
    {
      skuId: "APPLE-IPHONE-15-PRO-256",
      productId: "uuid-of-product",
      quantity: 1,
      snapshotPriceCents: 99900,    // price when added
      addedAt: ISODate("2026-03-20T14:30:00Z"),
      // Denormalized for quick rendering (avoids product service call)
      snapshotName: "iPhone 15 Pro 256GB",
      snapshotImageUrl: "https://cdn.example.com/products/iphone-15-pro.jpg",
      // Flags
      isFlashSaleItem: false,
      flashSaleId: null,
      maxQtyPerCart: 5
    }
  ],

  appliedCoupon: null,

  // Merge history (for debugging/support)
  mergeHistory: [
    {
      mergedAt: ISODate("2026-03-10T11:00:00Z"),
      sourceSessionId: "session-abc-123",
      itemsMerged: 2,
      strategy: "SMART_MERGE"
    }
  ]
}

// Indexes
db.carts.createIndex({ "userId": 1 }, { unique: true, sparse: true })
db.carts.createIndex({ "sessionId": 1 }, { sparse: true })
db.carts.createIndex({ "status": 1, "lastActivityAt": -1 })
db.carts.createIndex({ "lastActivityAt": 1 }, { expireAfterSeconds: 2592000 })  // TTL index
```

### PostgreSQL Cart Events (Audit Log)

```sql
-- Immutable append-only audit log — never updated, only inserted
CREATE TABLE cart_events (
    id              BIGSERIAL PRIMARY KEY,
    event_id        UUID UNIQUE NOT NULL DEFAULT gen_random_uuid(),
    cart_id         UUID NOT NULL,
    user_id         UUID,                    -- NULL for anonymous
    session_id      VARCHAR(128),            -- set for anonymous
    event_type      VARCHAR(30) NOT NULL,
                    -- ITEM_ADDED | ITEM_REMOVED | QTY_UPDATED |
                    -- COUPON_APPLIED | CART_MERGED | CART_EXPIRED |
                    -- CART_CHECKED_OUT | CART_ABANDONED
    sku_id          VARCHAR(100),
    old_qty         INT,
    new_qty         INT,
    price_cents     BIGINT,
    metadata        JSONB,                   -- flexible extra context
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT NOW()
)
PARTITION BY RANGE (occurred_at);

-- Monthly partitions
CREATE TABLE cart_events_2026_03 PARTITION OF cart_events
    FOR VALUES FROM ('2026-03-01') TO ('2026-04-01');
CREATE TABLE cart_events_2026_04 PARTITION OF cart_events
    FOR VALUES FROM ('2026-04-01') TO ('2026-05-01');

CREATE INDEX idx_cart_events_cart_id    ON cart_events(cart_id, occurred_at DESC);
CREATE INDEX idx_cart_events_user_id    ON cart_events(user_id, occurred_at DESC);
CREATE INDEX idx_cart_events_event_type ON cart_events(event_type, occurred_at DESC);
```

---

## 6. Redis Data Structures

```
Key Pattern                                    Type     TTL          Purpose
──────────────────────────────────────────────────────────────────────────────
cart:user:{userId}                             HASH     30d sliding  Logged-in cart
cart:session:{sessionId}                       HASH     7d fixed     Anonymous cart
cart:merge:{userId}:{sessionId}                STRING   1h           Merge idempotency
cart:version:{userId}                          STRING   30d          Optimistic lock version

flash:carted:{skuId}                           STRING   24h          Global cart count
flash:user:{userId}:{skuId}                    STRING   1h           Per-user flash limit
flash:rate:{skuId}:{second}                    STRING   2s           Per-SKU rate limiter

idempotency:add:{userId}:{skuId}:{requestId}   STRING   60s          Add-to-cart dedup
idempotency:remove:{userId}:{skuId}:{requestId} STRING  60s          Remove dedup

price:snapshot:{skuId}                         STRING   None         Used in flash sale lock

abandoned:notified:{userId}                    STRING   48h          Email throttle
```

---

## 7. Cart State Machine

```
                    ┌─────────────┐
                    │  ANONYMOUS  │  ← sessionId assigned, no login
                    │    CART     │
                    └──────┬──────┘
                           │ User logs in
                           │ triggerMerge()
                    ┌──────▼──────┐
         ┌──────────│   ACTIVE    │──────────┐
         │          │    CART     │          │
         │          └──────┬──────┘          │
         │                 │                 │
         │ No activity     │ User starts     │ 30-day
         │ 1+ day          │ checkout        │ TTL fires
         │                 │                 │
   ┌─────▼────┐    ┌───────▼──────┐   ┌──────▼──────┐
   │ ABANDONED│    │  IN_CHECKOUT │   │   EXPIRED   │
   │          │    │  (15min hold)│   │             │
   └─────┬────┘    └───────┬──────┘   └─────────────┘
         │                 │
         │ Remarketing     │ Payment success
         │ email sent      │
         │                 │
   ┌─────▼────┐    ┌───────▼──────┐
   │ Re-engage│    │ CHECKED_OUT  │
   │ → ACTIVE │    │  (archived)  │
   └──────────┘    └──────────────┘

Transitions:
  ANONYMOUS   → ACTIVE:       Login + merge
  ACTIVE      → ABANDONED:    1+ day no activity (soft flag)
  ACTIVE      → IN_CHECKOUT:  User initiates checkout
  ACTIVE      → EXPIRED:      30-day TTL fires
  ABANDONED   → ACTIVE:       User returns (re-engages)
  IN_CHECKOUT → ACTIVE:       Checkout abandoned or payment failed
  IN_CHECKOUT → CHECKED_OUT:  Payment succeeded
```

---

## 8. Kafka Event Flow

```
Topic               Partitions  Key         Retention   Consumers
──────────────────────────────────────────────────────────────────────────
cart-events         20          userId      7 days      cart-persister (→ MongoDB)
                                                        cart-analytics (→ PostgreSQL)
                                                        recommendation-service
                                                        abandoned-cart-emailer

cart-merged         5           userId      3 days      analytics
                                                        segment-tracker

cart-expired        5           userId      3 days      abandoned-cart-emailer
                                                        analytics

price-events        50          skuId       7 days      (from price-service)
                                                        cart-price-invalidator
```

**Event schemas:**

```java
// CartItemAddedEvent
record CartItemAddedEvent(
    String  eventId,        // UUID for idempotency
    String  userId,         // null if anonymous
    String  sessionId,      // null if logged-in
    String  cartId,
    String  skuId,
    int     previousQty,
    int     newQty,
    long    snapshotPriceCents,
    boolean isFlashSaleItem,
    Instant occurredAt
) {}

// CartMergedEvent
record CartMergedEvent(
    String  eventId,
    String  userId,
    String  sourceSessionId,
    List<MergedItem> mergedItems,     // items added from guest cart
    List<MergedItem> keptItems,       // items already in user cart
    List<ConflictResolution> conflicts,  // same-SKU resolutions
    String  mergeStrategy,            // "SMART_MERGE"
    Instant occurredAt
) {}

// CartExpiredEvent
record CartExpiredEvent(
    String      eventId,
    String      userId,
    String      cartId,
    List<String> skuIds,             // what was in the cart
    long        subtotalCents,       // for remarketing
    Instant     lastActivityAt,
    Instant     occurredAt
) {}
```

---

## 9. Implementation Code

### 9.1 Cart Service — Core Operations

```java
@RestController
@RequestMapping("/api/v1/cart")
@RequiredArgsConstructor
public class CartController {

    private final CartCommandService commandService;
    private final CartQueryService queryService;

    // Get current cart (enriched with live prices)
    @GetMapping
    public CartResponse getCart(
            @RequestHeader(value = "Authorization", required = false) String token,
            @RequestHeader(value = "X-Session-Id", required = false) String sessionId,
            @RequestParam(defaultValue = "en-US") String locale,
            @RequestParam(required = false) String storeId) {

        CartIdentity identity = resolveIdentity(token, sessionId);
        return queryService.getEnrichedCart(identity, locale, storeId);
    }

    // Add item to cart
    @PostMapping("/items")
    public CartResponse addItem(
            @RequestHeader(value = "Authorization", required = false) String token,
            @RequestHeader(value = "X-Session-Id", required = false) String sessionId,
            @RequestHeader("Idempotency-Key") String idempotencyKey,
            @RequestBody @Valid AddItemRequest request) {

        CartIdentity identity = resolveIdentity(token, sessionId);
        return commandService.addItem(identity, request, idempotencyKey);
    }

    // Update item quantity
    @PutMapping("/items/{skuId}")
    public CartResponse updateQuantity(
            @RequestHeader(value = "Authorization", required = false) String token,
            @RequestHeader(value = "X-Session-Id", required = false) String sessionId,
            @RequestHeader("Idempotency-Key") String idempotencyKey,
            @PathVariable String skuId,
            @RequestBody @Valid UpdateQtyRequest request) {

        CartIdentity identity = resolveIdentity(token, sessionId);
        return commandService.updateQuantity(identity, skuId, request.getQuantity(), idempotencyKey);
    }

    // Remove item
    @DeleteMapping("/items/{skuId}")
    public CartResponse removeItem(
            @RequestHeader(value = "Authorization", required = false) String token,
            @RequestHeader(value = "X-Session-Id", required = false) String sessionId,
            @PathVariable String skuId) {

        CartIdentity identity = resolveIdentity(token, sessionId);
        return commandService.removeItem(identity, skuId);
    }

    // Clear cart
    @DeleteMapping
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void clearCart(
            @RequestHeader(value = "Authorization", required = false) String token,
            @RequestHeader(value = "X-Session-Id", required = false) String sessionId) {

        CartIdentity identity = resolveIdentity(token, sessionId);
        commandService.clearCart(identity);
    }

    private CartIdentity resolveIdentity(String token, String sessionId) {
        if (token != null && token.startsWith("Bearer ")) {
            String userId = jwtService.extractUserId(token.substring(7));
            return CartIdentity.forUser(userId);
        }
        if (sessionId != null) {
            return CartIdentity.forSession(sessionId);
        }
        throw new CartIdentityException("No authentication or session ID provided");
    }
}
```

### 9.2 Anonymous Cart (Session-Based)

```java
@Service
@RequiredArgsConstructor
public class CartCommandService {

    private final StringRedisTemplate redis;
    private final RedisScript<List<Long>> addItemScript;  // Lua script bean
    private final KafkaTemplate<String, Object> kafka;
    private final CartMergeService mergeService;
    private final PriceService priceService;

    private static final long SESSION_CART_TTL_SECONDS   = 7L * 24 * 3600;   // 7 days
    private static final long USER_CART_TTL_SECONDS      = 30L * 24 * 3600;  // 30 days
    private static final int  DEFAULT_MAX_QTY_PER_ITEM   = 10;

    public CartResponse addItem(CartIdentity identity, AddItemRequest request, String idempotencyKey) {
        String cartKey = identity.toRedisKey();           // cart:user:{id} or cart:session:{id}
        long   ttl     = identity.isUser() ? USER_CART_TTL_SECONDS : SESSION_CART_TTL_SECONDS;

        // Fetch current snapshot price
        long snapshotPrice = priceService.getCurrentPrice(request.getSkuId(), request.getStoreId());

        // Build idempotency key
        String idemKey = "idempotency:add:" + identity.getId() + ":" +
                          request.getSkuId() + ":" + idempotencyKey;

        // Execute atomic Lua script (check qty limit + set fields + refresh TTL + idempotency)
        List<Long> result = redis.execute(addItemScript,
            List.of(cartKey, idemKey),
            request.getSkuId(),
            String.valueOf(request.getQuantity()),
            String.valueOf(request.getMaxQtyOverride() != null
                ? request.getMaxQtyOverride() : DEFAULT_MAX_QTY_PER_ITEM),
            "60",  // idempotency TTL seconds
            String.valueOf(snapshotPrice),
            String.valueOf(ttl)
        );

        long newQty   = result.get(0);
        long prevQty  = result.get(1);

        if (newQty == -2) {
            // Duplicate request — return current cart state (idempotent response)
            return getEnrichedCart(identity, "en-US", request.getStoreId());
        }
        if (newQty == -1) {
            throw new CartItemLimitException("Maximum quantity of "
                + DEFAULT_MAX_QTY_PER_ITEM + " per item reached");
        }

        // Publish event (async persistence to MongoDB)
        String userId = identity.isUser() ? identity.getId() : null;
        kafka.send("cart-events", identity.getId(),
            new CartItemAddedEvent(
                UUID.randomUUID().toString(),
                userId,
                identity.isUser() ? null : identity.getId(),
                generateCartId(identity),
                request.getSkuId(),
                (int) prevQty,
                (int) newQty,
                snapshotPrice,
                false,
                Instant.now()
            ));

        return getEnrichedCart(identity, "en-US", request.getStoreId());
    }

    public CartResponse removeItem(CartIdentity identity, String skuId) {
        String cartKey = identity.toRedisKey();

        // Remove all fields for this SKU from the hash
        redis.opsForHash().delete(cartKey,
            skuId + ":qty",
            skuId + ":snapshotPrice",
            skuId + ":addedAt",
            skuId + ":metadata"
        );

        // Refresh TTL
        redis.expire(cartKey, Duration.ofSeconds(
            identity.isUser() ? USER_CART_TTL_SECONDS : SESSION_CART_TTL_SECONDS));

        kafka.send("cart-events", identity.getId(),
            new CartItemRemovedEvent(identity.getId(), skuId, Instant.now()));

        return getEnrichedCart(identity, "en-US", null);
    }
}
```

### 9.3 Cart Merge Strategy

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class CartMergeService {

    private final StringRedisTemplate redis;
    private final RedisScript<Long> mergeScript;   // atomic Lua for merge
    private final KafkaTemplate<String, Object> kafka;

    /**
     * Called when an anonymous user logs in.
     * Merges session cart into user cart using Smart Merge strategy.
     */
    @Transactional  // logical — not DB transaction, used for error rollback logging
    public MergeResult mergeOnLogin(String userId, String sessionId) {
        // Idempotency: prevent double-merge
        String mergeIdempotencyKey = "cart:merge:" + userId + ":" + sessionId;
        Boolean firstMerge = redis.opsForValue()
            .setIfAbsent(mergeIdempotencyKey, "done", Duration.ofHours(1));

        if (Boolean.FALSE.equals(firstMerge)) {
            log.info("Cart merge already done for userId={} sessionId={}", userId, sessionId);
            return MergeResult.alreadyMerged();
        }

        String sessionKey = "cart:session:" + sessionId;
        String userKey    = "cart:user:" + userId;

        // Read both carts
        Map<Object, Object> sessionEntries = redis.opsForHash().entries(sessionKey);
        Map<Object, Object> userEntries    = redis.opsForHash().entries(userKey);

        if (sessionEntries.isEmpty()) {
            // No guest cart — nothing to merge
            return MergeResult.noGuestCart();
        }

        // Parse items from hash entries
        Map<String, CartItem> guestItems = parseCartItems(sessionEntries);
        Map<String, CartItem> userItems  = parseCartItems(userEntries);

        // Apply Smart Merge strategy
        SmartMergeResult merge = applySmartMerge(guestItems, userItems);

        // Atomic merge execution via Lua
        List<String> newFields = buildRedisFields(merge.getMergedItems());
        redis.execute(mergeScript,
            List.of(sessionKey, userKey),
            newFields.toArray(String[]::new)
        );

        // Reset user cart TTL
        redis.expire(userKey, Duration.ofDays(30));

        // Publish event for MongoDB sync + analytics
        kafka.send("cart-merged", userId,
            new CartMergedEvent(
                UUID.randomUUID().toString(),
                userId,
                sessionId,
                merge.getAddedItems(),
                merge.getKeptItems(),
                merge.getConflicts(),
                "SMART_MERGE",
                Instant.now()
            ));

        log.info("Cart merged: userId={}, guestItems={}, userItems={}, mergedTotal={}",
            userId, guestItems.size(), userItems.size(), merge.getMergedItems().size());

        return MergeResult.success(merge);
    }

    private SmartMergeResult applySmartMerge(
            Map<String, CartItem> guestItems,
            Map<String, CartItem> userItems) {

        Map<String, CartItem> merged = new LinkedHashMap<>(userItems);  // start with user cart
        List<MergedItem> addedFromGuest = new ArrayList<>();
        List<ConflictResolution> conflicts = new ArrayList<>();

        for (Map.Entry<String, CartItem> guestEntry : guestItems.entrySet()) {
            String skuId = guestEntry.getKey();
            CartItem guestItem = guestEntry.getValue();

            if (!merged.containsKey(skuId)) {
                // SKU only in guest cart → ADD to merged
                merged.put(skuId, guestItem);
                addedFromGuest.add(new MergedItem(skuId, 0, guestItem.getQty(), "ADDED"));
            } else {
                // SKU in both → SMART MERGE: take MAX quantity
                CartItem userItem = merged.get(skuId);
                int resolvedQty = Math.max(guestItem.getQty(), userItem.getQty());
                // Keep guest snapshot price if guest item was more recently priced
                long resolvedPrice = guestItem.getAddedAt().isAfter(userItem.getAddedAt())
                    ? guestItem.getSnapshotPrice() : userItem.getSnapshotPrice();

                merged.put(skuId, userItem.withQty(resolvedQty).withSnapshotPrice(resolvedPrice));
                conflicts.add(new ConflictResolution(skuId, guestItem.getQty(),
                    userItem.getQty(), resolvedQty, "MAX_QTY"));
            }
        }

        return new SmartMergeResult(merged, addedFromGuest,
            new ArrayList<>(userItems.values()), conflicts);
    }
}
```

### 9.4 Price Snapshot & Live Price Enrichment

```java
@Service
@RequiredArgsConstructor
public class CartQueryService {

    private final StringRedisTemplate redis;
    private final PriceServiceClient priceClient;
    private final ProductServiceClient productClient;
    private final CartMergeService mergeService;

    public CartResponse getEnrichedCart(CartIdentity identity, String locale, String storeId) {
        String cartKey = identity.toRedisKey();

        // 1. Fetch raw cart from Redis
        Map<Object, Object> rawCart = redis.opsForHash().entries(cartKey);

        if (rawCart.isEmpty()) {
            // Redis miss for logged-in user → try MongoDB fallback
            if (identity.isUser()) {
                return rebuildFromMongoDB(identity.getId());
            }
            return CartResponse.empty();
        }

        // Refresh TTL on every read (sliding window)
        long ttl = identity.isUser() ? 30L * 24 * 3600 : 7L * 24 * 3600;
        redis.expire(cartKey, Duration.ofSeconds(ttl));

        // 2. Parse cart items from hash
        Map<String, CartItem> items = parseCartItems(rawCart);

        if (items.isEmpty()) {
            return CartResponse.empty();
        }

        // 3. Batch-fetch current prices (single Redis pipeline call)
        Map<String, Long> currentPrices = fetchCurrentPricesBatch(
            items.keySet(), storeId);

        // 4. Enrich each item with live price + compute alerts
        List<CartItemResponse> enrichedItems = items.entrySet().stream()
            .map(entry -> {
                String  skuId         = entry.getKey();
                CartItem item         = entry.getValue();
                long    currentPrice  = currentPrices.getOrDefault(skuId, item.getSnapshotPrice());
                long    priceDelta    = currentPrice - item.getSnapshotPrice();
                String  priceAlert    = computePriceAlert(priceDelta, item.getSnapshotPrice());

                return CartItemResponse.builder()
                    .skuId(skuId)
                    .quantity(item.getQty())
                    .snapshotPriceCents(item.getSnapshotPrice())
                    .currentPriceCents(currentPrice)
                    .priceDeltaCents(priceDelta)
                    .priceDeltaPct(priceDelta * 100.0 / item.getSnapshotPrice())
                    .priceAlert(priceAlert)
                    .addedAt(item.getAddedAt())
                    .subtotalCents(currentPrice * item.getQty())
                    .build();
            })
            .collect(Collectors.toList());

        // 5. Compute totals
        long subtotal  = enrichedItems.stream().mapToLong(CartItemResponse::getSubtotalCents).sum();
        long savings   = enrichedItems.stream()
            .mapToLong(i -> (i.getSnapshotPriceCents() - i.getCurrentPriceCents()) * i.getQuantity())
            .sum();

        return CartResponse.builder()
            .cartId(generateCartId(identity))
            .items(enrichedItems)
            .subtotalCents(subtotal)
            .savingsVsSnapshotCents(savings)
            .itemCount(enrichedItems.stream().mapToInt(CartItemResponse::getQuantity).sum())
            .hasPriceAlerts(enrichedItems.stream().anyMatch(i -> i.getPriceAlert() != null))
            .lastUpdatedAt(getLastActivityAt(rawCart))
            .build();
    }

    private Map<String, Long> fetchCurrentPricesBatch(Set<String> skuIds, String storeId) {
        // Use Redis pipeline to fetch all prices in one round trip
        List<Object> prices = redis.executePipelined(connection -> {
            for (String skuId : skuIds) {
                String field = storeId != null ? storeId : "NATIONAL";
                connection.hashCommands().hGet(
                    ("price:" + skuId).getBytes(),
                    field.getBytes()
                );
            }
            return null;
        });

        Map<String, Long> result = new LinkedHashMap<>();
        int i = 0;
        for (String skuId : skuIds) {
            Object price = prices.get(i++);
            if (price instanceof byte[] bytes) {
                result.put(skuId, Long.parseLong(new String(bytes)));
            }
        }
        return result;
    }

    private String computePriceAlert(long priceDelta, long snapshotPrice) {
        if (snapshotPrice == 0) return null;
        double pct = Math.abs(priceDelta * 100.0 / snapshotPrice);
        if (pct < 1.0) return null;           // less than 1% change — no alert
        return priceDelta < 0 ? "PRICE_DROPPED" : "PRICE_INCREASED";
    }
}
```

### 9.5 Flash Sale Race-Condition Guard (Lua Script as Spring Bean)

```java
@Configuration
public class RedisScriptConfig {

    /**
     * Atomic add-to-cart Lua script.
     * Handles: idempotency, quantity limit, TTL refresh — all in one atomic operation.
     */
    @Bean
    public RedisScript<List<Long>> addItemScript() {
        String script = """
            -- KEYS[1] = cart key
            -- KEYS[2] = idempotency key
            -- ARGV[1] = skuId
            -- ARGV[2] = deltaQty
            -- ARGV[3] = maxQty
            -- ARGV[4] = idemTTL (seconds)
            -- ARGV[5] = snapshotPrice
            -- ARGV[6] = cartTTL (seconds)
            -- Returns: {newQty, prevQty} or {-1, currentQty} if limit exceeded or {-2, 0} if duplicate

            local idemSet = redis.call('SET', KEYS[2], '1', 'NX', 'EX', ARGV[4])
            if idemSet == false then
                return {-2, 0}
            end

            local currentQty = tonumber(redis.call('HGET', KEYS[1], ARGV[1] .. ':qty') or '0')
            local deltaQty   = tonumber(ARGV[2])
            local newQty     = currentQty + deltaQty

            if newQty > tonumber(ARGV[3]) then
                return {-1, currentQty}
            end

            redis.call('HSET', KEYS[1],
                ARGV[1] .. ':qty',           tostring(newQty),
                ARGV[1] .. ':snapshotPrice', ARGV[5],
                ARGV[1] .. ':addedAt',       tostring(redis.call('TIME')[1]),
                '__lastActivityAt',          tostring(redis.call('TIME')[1])
            )
            redis.call('EXPIRE', KEYS[1], ARGV[6])

            return {newQty, currentQty}
            """;
        return RedisScript.of(script, (Class<List<Long>>) (Class<?>) List.class);
    }

    /**
     * Atomic cart merge Lua script.
     * Deletes session cart and writes merged result to user cart atomically.
     */
    @Bean
    public RedisScript<Long> mergeScript() {
        String script = """
            -- KEYS[1] = session cart key (to delete)
            -- KEYS[2] = user cart key (to overwrite)
            -- ARGV = alternating field-value pairs for the merged cart

            redis.call('DEL', KEYS[1])
            redis.call('DEL', KEYS[2])

            if #ARGV > 0 then
                redis.call('HSET', KEYS[2], unpack(ARGV))
                redis.call('EXPIRE', KEYS[2], 2592000)
            end

            return 1
            """;
        return RedisScript.of(script, Long.class);
    }
}
```

### 9.6 Cart Expiry — TTL + Sliding Window

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class CartExpiryService {

    private final StringRedisTemplate redis;
    private final KafkaTemplate<String, Object> kafka;

    /**
     * Refreshes the TTL sliding window on every cart read or write.
     * Called automatically by CartCommandService and CartQueryService.
     */
    public void refreshTTL(CartIdentity identity) {
        String cartKey = identity.toRedisKey();
        long   ttlSecs = identity.isUser()
            ? 30L * 24 * 3600   // 30 days for users
            : 7L  * 24 * 3600;  // 7 days for sessions

        redis.expire(cartKey, Duration.ofSeconds(ttlSecs));

        // Update lastActivityAt field
        redis.opsForHash().put(cartKey, "__lastActivityAt",
            String.valueOf(Instant.now().getEpochSecond()));
    }

    /**
     * Listen for Redis keyspace expiry notifications.
     * Redis must be configured: notify-keyspace-events "Ex"
     * This fires when a cart:user:* or cart:session:* key expires.
     */
    @Bean
    public MessageListenerAdapter cartExpiredListener() {
        return new MessageListenerAdapter(new RedisKeyExpiryListener(kafka));
    }

    @Bean
    public RedisMessageListenerContainer keyExpiryContainer(
            RedisConnectionFactory factory,
            MessageListenerAdapter listener) {

        RedisMessageListenerContainer container = new RedisMessageListenerContainer();
        container.setConnectionFactory(factory);
        container.addMessageListener(listener,
            new PatternTopic("__keyevent@0__:expired"));
        return container;
    }
}

// Handles Redis keyspace expiration events
@RequiredArgsConstructor
@Slf4j
class RedisKeyExpiryListener implements MessageListener {

    private final KafkaTemplate<String, Object> kafka;

    @Override
    public void onMessage(Message message, byte[] pattern) {
        String expiredKey = new String(message.getBody());

        if (expiredKey.startsWith("cart:user:")) {
            String userId = expiredKey.substring("cart:user:".length());
            log.info("Cart expired for userId={}", userId);

            // Publish expiry event → remarketing, analytics
            kafka.send("cart-expired", userId,
                new CartExpiredEvent(
                    UUID.randomUUID().toString(),
                    userId,
                    expiredKey,
                    List.of(),   // can't read the key — it's already gone
                    0L,
                    Instant.now(),
                    Instant.now()
                ));
        }
    }
}
```

### 9.7 Cart Persistence Worker (Redis → MongoDB)

```java
// Kafka consumer: persists cart state to MongoDB after every change
@Component
@RequiredArgsConstructor
@Slf4j
public class CartPersistenceWorker {

    private final MongoCartRepository mongoRepo;
    private final StringRedisTemplate redis;

    @KafkaListener(
        topics = "cart-events",
        groupId = "cart-persister",
        concurrency = "5"   // 5 parallel consumers
    )
    public void persistCartChange(CartItemAddedEvent event) {
        try {
            String userId = event.getUserId();
            if (userId == null) return;  // don't persist anonymous carts to MongoDB

            // Read current cart state from Redis
            String cartKey = "cart:user:" + userId;
            Map<Object, Object> rawCart = redis.opsForHash().entries(cartKey);

            if (rawCart.isEmpty()) {
                log.warn("Cart not found in Redis for userId={}, skipping MongoDB sync", userId);
                return;
            }

            // Build MongoDB document
            Cart mongoCart = buildMongoCart(userId, rawCart);

            // Upsert (replace or insert)
            mongoRepo.save(mongoCart);

            log.debug("Cart persisted to MongoDB for userId={}, items={}",
                userId, mongoCart.getItems().size());

        } catch (Exception e) {
            log.error("Failed to persist cart for userId={}: {}", event.getUserId(), e.getMessage());
            // Kafka will retry (at-least-once delivery)
        }
    }

    @KafkaListener(topics = "cart-merged", groupId = "cart-persister-merge", concurrency = "3")
    public void persistMergedCart(CartMergedEvent event) {
        // After merge, read the final merged cart from Redis and persist
        String cartKey = "cart:user:" + event.getUserId();
        Map<Object, Object> rawCart = redis.opsForHash().entries(cartKey);

        if (!rawCart.isEmpty()) {
            Cart mongoCart = buildMongoCart(event.getUserId(), rawCart);
            mongoCart.getMergeHistory().add(new MergeHistoryEntry(
                Instant.now(),
                event.getSourceSessionId(),
                event.getMergedItems().size(),
                event.getMergeStrategy()
            ));
            mongoRepo.save(mongoCart);
        }
    }

    private Cart buildMongoCart(String userId, Map<Object, Object> rawCart) {
        Map<String, CartItem> items = parseCartItems(rawCart);

        return Cart.builder()
            .userId(userId)
            .status(CartStatus.ACTIVE)
            .version(Long.parseLong(rawCart.getOrDefault("__version", "1").toString()))
            .lastActivityAt(Instant.ofEpochSecond(
                Long.parseLong(rawCart.getOrDefault("__lastActivityAt", "0").toString())))
            .expiresAt(Instant.now().plus(Duration.ofDays(30)))
            .items(items.values().stream()
                .map(item -> CartDocument.Item.builder()
                    .skuId(item.getSkuId())
                    .quantity(item.getQty())
                    .snapshotPriceCents(item.getSnapshotPrice())
                    .addedAt(item.getAddedAt())
                    .build())
                .collect(Collectors.toList()))
            .build();
    }
}
```

---

## 10. Failure Scenarios & Mitigations

### Scenario 1: Redis Crash (Cart Data Lost)

```
Problem: Redis node fails mid-session → user's cart disappears
         Without MongoDB backup: user loses everything they added

Mitigation:
  1. Redis Sentinel (3 nodes) or Redis Cluster → auto-failover <30s
  2. MongoDB as durable backup:
     - Cart persisted asynchronously after every change via Kafka
     - On Redis miss for logged-in user: rebuild cart from MongoDB
     - At most 1 Kafka message lag of data loss (<1 second)

  Recovery path (cart-query-service):
    Redis MISS → MongoDB findOne({userId}) → if found:
      Rebuild Redis key from MongoDB doc:
        HSET cart:user:{userId} ...all fields from MongoDB
        EXPIRE cart:user:{userId} 2592000
      Return rebuilt cart to user (transparent recovery)

  Anonymous carts (session-based): NOT backed up to MongoDB
    Policy: anonymous cart loss is acceptable (7-day ephemeral data)
    On loss: user sees empty cart, re-adds items
```

### Scenario 2: Kafka Lag — MongoDB Sync Falls Behind

```
Problem: Burst of 100,000 cart events → Kafka consumer lag → MongoDB not in sync
         Redis cart is authoritative but MongoDB is hours behind

Impact: Low — MongoDB is only used for Redis miss recovery
        Normal read path never touches MongoDB

Mitigation:
  1. Monitor consumer lag: alert if lag > 10,000 messages (>10 seconds behind)
  2. Scale cart-persister consumer group: up to 20 instances
  3. MongoDB writes are idempotent (upsert by userId) → safe to process out of order
  4. For critical recovery: fallback to PostgreSQL cart_events replay
     Re-read all cart_events for userId, reconstruct cart state from events
```

### Scenario 3: Race Condition in Merge (User Logs In Twice Simultaneously)

```
Problem: User opens two tabs, both trigger login simultaneously
         Both initiate cart merge → one merge overwrites the other's result

Mitigation:
  Idempotency key: "cart:merge:{userId}:{sessionId}" with SETNX
  First merge acquires the key → proceeds
  Second merge finds key exists → returns already-merged cart
  Guaranteed by Redis SETNX atomicity (only one winner)
```

### Scenario 4: Price Service Unavailable During Cart Read

```
Problem: Price-service is down → can't fetch current prices
         Cart read fails → users can't see their cart

Mitigation:
  1. Circuit breaker (Resilience4j) around price service calls
  2. On circuit open: fall back to snapshotPrice for display
     Items show snapshotPrice with banner: "Prices may be out of date"
  3. Cart operations (add/remove) still work; only live enrichment is degraded
  4. Price calls use short timeout (100ms) + fallback immediately

  @CircuitBreaker(name = "price-service", fallbackMethod = "getSnapshotPrice")
  public long getCurrentPrice(String skuId, String storeId) { ... }

  public long getSnapshotPrice(String skuId, String storeId, Throwable t) {
      return snapshotPriceFromCart;   // use the stored snapshot
  }
```

### Scenario 5: Flash Sale — Inventory Inconsistency

```
Problem: 100,000 users add item to cart, but only 50 units exist
         Cart additions don't check inventory (by design — reservation at checkout)
         All 100,000 users have item in cart → at checkout, 99,950 get "sold out"

Design decision: This is intentional — cart is NOT an inventory reservation.
  Pro: No contention at add-to-cart; better UX for browsing
  Con: User disappointment at checkout

Mitigation:
  1. Per-cart qty limit per SKU (Lua script enforces max 3)
  2. Display "Limited stock" badge when inventory < 50 units
     Read from Redis inventory key (set by inventory-service)
     "low_stock:{skuId}" → "15" (15 remaining)
  3. At checkout: inventory reservation (see Question 3 design)
  4. Real-time availability signal:
     availability:{skuId} → "IN_STOCK" | "LOW_STOCK" | "OUT_OF_STOCK"
     Show badge on cart page: "Only 5 left!"
     Updated by inventory-service via Kafka → Redis
```

---

## 11. Scaling Strategy

### Horizontal Scaling

```
Service               Instances    Scale Trigger                 Notes
──────────────────────────────────────────────────────────────────────────
cart-service          5-50         Cart ops/sec > 50K            Stateless
cart-persister        5-20         Kafka consumer lag > 5K       Kafka consumer group
Redis                 6 nodes      Memory > 70% or ops > 200K    Cluster, 3P+3R
MongoDB               3 nodes      Write latency > 50ms          RS: 1P+2S
PostgreSQL            1P+2R        Query latency > 100ms         Events only (low write)
Kafka                 6 brokers    Consumer lag growing          20 partitions
```

### Redis Memory Sizing

```
Active user carts:    50,000 × 2 KB  = 100 MB
30-day user carts:    500,000 × 2 KB = 1 GB    (assuming 10x daily active base)
Session carts:        20,000 × 1 KB  = 20 MB
Idempotency keys:     100K × 50 B    = 5 MB
Flash rate limiters:  10K × 20 B     = 0.2 MB
Price cache:          500K × 100 B   = 50 MB   (shared with price-service)
──────────────────────────────────────────────
Total:                ~1.2 GB

With 3× safety buffer: 4 GB Redis memory → trivially fits on any Redis node
Redis Cluster nodes: 3 master × 4 GB = 12 GB total capacity → well headroom
```

### Throughput Capacity

```
Redis throughput:
  Normal peak: 250,000 cart ops/sec
  Flash sale:  100,000 add-to-cart/sec (burst)
  Redis Cluster (3 nodes): ~300,000 ops/sec total → sufficient

  For flash sale burst: add token bucket (5,000 add-to-cart/sec per SKU)
  Excess requests → 429 Too Many Requests with Retry-After header

Cart-service throughput:
  Each instance: ~5,000 req/sec (Spring Boot + Lettuce async Redis)
  50 instances → 250,000 req/sec capacity
  Autoscale when p99 latency > 50ms
```

---

## 12. Monitoring & Observability

### Key Metrics

```
cart_add_total{result}                  → total adds: success | limit_exceeded | sold_out
cart_add_latency_p99                    → target <30ms
cart_read_latency_p99                   → target <50ms
cart_merge_total{result}                → successful merges per minute
cart_merge_latency_p99                  → target <100ms

redis_cart_cache_hit_rate               → should be ~100% (carts always in Redis)
redis_cart_miss_total                   → triggers MongoDB rebuild; should be rare
redis_cart_rebuild_from_mongo_total     → recovery metric

cart_expired_total                      → daily expiry volume
cart_abandoned_total                    → abandonment rate (business KPI)
cart_checked_out_conversion_rate        → cart → checkout conversion (business KPI)

kafka_consumer_lag{group=cart-persister} → target <1,000
price_enrichment_fallback_total         → circuit breaker fires (price-service down)

flash_sale_rate_limited_total           → add-to-cart requests rejected during flash sale
flash_sale_cart_adds_per_second         → burst monitor
```

### Alerts

```
CRITICAL:
  redis_cart_miss_rate > 5%               → Redis evicting carts! Check memory
  cart_add_latency_p99 > 500ms            → Redis or app performance issue
  kafka_consumer_lag{cart-persister} > 50K → MongoDB sync severely delayed

WARNING:
  price_enrichment_fallback_rate > 1%     → Price service degraded
  cart_merge_failures_total > 10/min      → Login merge issues
  cart_add_latency_p99 > 100ms            → Scale cart-service pods
  redis_memory_used_pct > 80%             → Increase Redis cluster size
```

---

## 13. Summary Cheat Sheet

### The 5 Core Answers at a Glance

| Question | Answer | Key Technology |
|---|---|---|
| Where to store cart data? | Redis as hot store (Hash per cart, TTL managed); MongoDB as durable backup for logged-in users; PostgreSQL for immutable audit events only | Redis Hash + MongoDB |
| How to handle cart merge? | Smart Merge: union of all items, max-quantity for conflicts; Lua atomic delete-session + write-user; idempotency key prevents double-merge | Redis Lua + SETNX |
| How to handle price changes? | Dual-price model: store snapshotPrice at add-time, enrich with currentPrice on every read; alert user on >1% delta; force re-confirm at checkout if price changed | Redis Hash HGET (live price) |
| Race conditions during flash sales? | Lua script for atomic qty check-and-set; Redisson distributed lock for global cart limits; token bucket rate limiter per SKU | Redis Lua + Redisson |
| Cart expiration? | Redis TTL reset on every read/write (sliding window = 30 days from last activity); Redis keyspace events trigger Kafka expiry event; MongoDB TTL index auto-deletes stale documents | Redis EXPIRE + Keyspace events |

### Golden Rules for Cart Systems

```
1.  Redis is the primary cart store — never the DB; DB is the durable backup
2.  Lua scripts for all quantity checks — never read-then-write without atomicity
3.  Dual-price model: snapshotPrice stored, currentPrice fetched — never update stored price
4.  Cart merge must be idempotent — retries and race conditions are inevitable
5.  Smart merge strategy: add guest items, keep user items, take MAX on conflicts
6.  Sliding TTL window: reset expiry on EVERY cart access — not just writes
7.  Redis keyspace events: listen for key expiry → trigger remarketing pipelines
8.  Cart ≠ inventory reservation — add-to-cart never decrements stock
9.  Price enrichment is read-time, not write-time — avoids stale cart updates from price events
10. Anonymous cart is ephemeral (Redis only) — logged-in cart is durable (Redis + MongoDB)
11. Circuit breaker around price-service — cart reads must never fail due to a price lookup
12. Kafka async persistence — cart writes to MongoDB are eventually consistent (acceptable)
```

### Technology Role Summary

```
Redis:        ★ Primary cart store (Hash per userId/sessionId)
              ★ TTL-based sliding expiry (EXPIRE on every access)
              ★ Atomic add/merge operations (Lua scripts)
              ★ Per-SKU rate limiting during flash sales
              ★ Idempotency keys for dedup
              ★ Keyspace expiry notifications

Kafka:        ★ Async cart persistence (Redis → MongoDB)
              ★ Cart event stream (add/remove/merge/expire/checkout)
              ★ Abandoned cart trigger → email remarketing
              ★ Price change fan-out → cart invalidation

MongoDB:      ★ Durable cart backup for logged-in users
              ★ Cart recovery source on Redis miss
              ★ Merge history and audit trail
              ★ TTL index for auto-expiry of old carts
              ★ Abandoned cart repository for remarketing

PostgreSQL:   ★ Immutable append-only cart events audit log
              ★ Partitioned by month for archival
              ★ Coupon/promo usage tracking (uniqueness enforced)

Spring Boot:  ★ Lua script execution (atomic cart operations)
              ★ Cart merge logic (Smart Merge strategy)
              ★ Price enrichment with circuit breaker fallback
              ★ Redis keyspace event listener (cart expiry)
              ★ Kafka consumers (persistence worker, price invalidator)
```
