# Question 5: Inventory Management Across Multiple Warehouses

## Scenario

Design an inventory system for a retailer with:
- **50 warehouses** across the US
- **100,000 SKUs**
- Real-time inventory tracking (sales, restocks, transfers, returns)
- When a customer orders, the system must **find the optimal warehouse** (closest, in-stock, lowest shipping cost)
- Support **inventory transfers between warehouses**
- **Daily reconciliation** with physical counts
- Handle **inventory holds** for pending orders

---

## Design Questions

1. How do you structure inventory data (centralized vs distributed)?
2. How do you handle concurrent inventory updates?
3. How do you implement the warehouse selection algorithm?
4. How do you track inventory holds vs available inventory?
5. How do you handle inventory discrepancies found during reconciliation?
6. How do you ensure inventory accuracy at high transaction volumes?

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          API Gateway                                    │
└────────────┬──────────────┬──────────────┬──────────────────────────────┘
             │              │              │
    ┌────────▼───────┐ ┌────▼─────┐ ┌─────▼──────────────┐
    │  Inventory     │ │Warehouse │ │  Reconciliation     │
    │  Service       │ │ Router   │ │  Service            │
    │  (writes)      │ │ Service  │ │                     │
    └────────┬───────┘ └────┬─────┘ └─────┬──────────────┘
             │              │             │
    ┌────────▼──────────────▼─────────────▼──────────────┐
    │                    Kafka                            │
    │  inventory.movements  warehouse.selections          │
    │  inventory.holds      inventory.transfers           │
    │  inventory.reconcile  inventory.alerts              │
    └────────┬───────────────────────────────────────────┘
             │
    ┌────────▼───────────────────────────────────────────┐
    │              Data Tier                              │
    │                                                     │
    │  PostgreSQL          Redis               MongoDB    │
    │  (ACID writes)       (speed layer)       (audit)   │
    │  inventory_levels    inv:{w}:{sku}       movements  │
    │  holds               available:{w}:{s}  snapshots  │
    │  transfers           warehouse_meta      discrepan  │
    └────────────────────────────────────────────────────┘
```

---

## Microservices Breakdown

| Service | Responsibility | Owns |
|---|---|---|
| **inventory-service** | Write all inventory movements (sales, restocks, returns, adjustments) | PostgreSQL `inventory_levels`, `inventory_holds`, `outbox_events` |
| **warehouse-router-service** | Warehouse selection algorithm for orders | Redis warehouse metadata, scoring cache |
| **transfer-service** | Initiate and track inventory transfers between warehouses | PostgreSQL `transfers` |
| **reconciliation-service** | Daily physical count comparison, discrepancy resolution | MongoDB `reconciliation_sessions`, `discrepancies` |
| **inventory-query-service** | Read-only queries (available stock, history) | Redis (primary), PostgreSQL read replica (fallback) |
| **alert-service** | Low stock alerts, anomaly detection | Redis threshold checks, Kafka consumer |

---

## Question 1: Centralized vs Distributed Inventory Data Structure

### Decision: Centralized PostgreSQL + Redis Speed Layer

**Option A – Distributed per warehouse (Rejected)**
```
Each warehouse has its own database shard.
Query: "Is SKU-123 available in ANY warehouse near zip 10001?"
→ Must query all 50 shards and merge → N+1 query problem
→ Cross-warehouse aggregate queries are expensive
→ Split-brain: global oversell risk between warehouse shards
```

**Option B – Centralized PostgreSQL + Redis Speed Layer (Chosen)**
```
One PostgreSQL cluster (primary + replicas) owns all inventory.
Redis layer holds pre-computed available stock per warehouse+SKU.

Reads:  Redis lookup → sub-millisecond
Writes: PostgreSQL ACID → Redis async sync
Cross-warehouse queries: single SQL query with GROUP BY warehouse_id

Trade-off: single DB cluster is the write bottleneck.
Mitigation: PostgreSQL partitioning by warehouse_id, read replicas,
            Redis absorbs ~95% of read traffic.
```

### PostgreSQL Schema

```sql
-- Core inventory table (source of truth)
-- Partitioned by warehouse_id for query isolation
CREATE TABLE inventory_levels (
    id              BIGSERIAL,
    warehouse_id    SMALLINT        NOT NULL,  -- 1–50
    sku_id          VARCHAR(50)     NOT NULL,
    quantity_on_hand INTEGER        NOT NULL DEFAULT 0 CHECK (quantity_on_hand >= 0),
    quantity_on_hold INTEGER        NOT NULL DEFAULT 0 CHECK (quantity_on_hold >= 0),
    quantity_in_transit INTEGER     NOT NULL DEFAULT 0,
    reorder_point   INTEGER         NOT NULL DEFAULT 0,
    reorder_quantity INTEGER        NOT NULL DEFAULT 0,
    version         BIGINT          NOT NULL DEFAULT 0,  -- optimistic locking
    last_movement_at TIMESTAMPTZ    NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ     NOT NULL DEFAULT now(),
    PRIMARY KEY (id, warehouse_id)
) PARTITION BY LIST (warehouse_id);

-- Each warehouse gets its own partition
CREATE TABLE inventory_levels_w01 PARTITION OF inventory_levels
    FOR VALUES IN (1);
-- ... repeat for all 50 warehouses

-- Composite unique index: one record per warehouse+SKU
CREATE UNIQUE INDEX idx_inv_warehouse_sku
    ON inventory_levels (warehouse_id, sku_id);

-- Fast lookup for available stock (on_hand - on_hold)
CREATE INDEX idx_inv_available
    ON inventory_levels (warehouse_id, sku_id)
    WHERE quantity_on_hand > quantity_on_hold;

-- Holds table (pending order reservations)
CREATE TABLE inventory_holds (
    hold_id         UUID            PRIMARY KEY DEFAULT gen_random_uuid(),
    warehouse_id    SMALLINT        NOT NULL,
    sku_id          VARCHAR(50)     NOT NULL,
    order_id        VARCHAR(100)    NOT NULL,
    quantity        INTEGER         NOT NULL CHECK (quantity > 0),
    hold_status     VARCHAR(20)     NOT NULL DEFAULT 'ACTIVE',
                                    -- ACTIVE, CONFIRMED, RELEASED, EXPIRED
    expires_at      TIMESTAMPTZ     NOT NULL,
    created_at      TIMESTAMPTZ     NOT NULL DEFAULT now(),
    confirmed_at    TIMESTAMPTZ,
    released_at     TIMESTAMPTZ
);

CREATE INDEX idx_holds_sku_warehouse   ON inventory_holds (warehouse_id, sku_id) WHERE hold_status = 'ACTIVE';
CREATE INDEX idx_holds_order           ON inventory_holds (order_id);
CREATE INDEX idx_holds_expires         ON inventory_holds (expires_at) WHERE hold_status = 'ACTIVE';

-- Every stock movement — immutable audit log
CREATE TABLE inventory_movements (
    id              BIGSERIAL       PRIMARY KEY,
    warehouse_id    SMALLINT        NOT NULL,
    sku_id          VARCHAR(50)     NOT NULL,
    movement_type   VARCHAR(30)     NOT NULL,
                                    -- SALE, RESTOCK, RETURN, TRANSFER_OUT,
                                    -- TRANSFER_IN, ADJUSTMENT, CYCLE_COUNT
    quantity_delta  INTEGER         NOT NULL,  -- negative=decrease, positive=increase
    quantity_after  INTEGER         NOT NULL,  -- snapshot after movement
    reference_id    VARCHAR(100),              -- order_id, transfer_id, etc.
    initiated_by    VARCHAR(100),              -- user or system
    notes           TEXT,
    occurred_at     TIMESTAMPTZ     NOT NULL DEFAULT now()
);

CREATE INDEX idx_movements_warehouse_sku ON inventory_movements (warehouse_id, sku_id, occurred_at DESC);
CREATE INDEX idx_movements_reference     ON inventory_movements (reference_id);

-- Warehouse master data
CREATE TABLE warehouses (
    warehouse_id    SMALLINT        PRIMARY KEY,
    warehouse_code  VARCHAR(20)     NOT NULL UNIQUE,
    name            VARCHAR(100)    NOT NULL,
    city            VARCHAR(100)    NOT NULL,
    state           CHAR(2)         NOT NULL,
    zip_code        VARCHAR(10)     NOT NULL,
    latitude        DECIMAL(9, 6)   NOT NULL,
    longitude       DECIMAL(9, 6)   NOT NULL,
    shipping_zone   SMALLINT        NOT NULL,  -- 1-8 UPS zones from this warehouse
    is_active       BOOLEAN         NOT NULL DEFAULT true,
    priority_rank   SMALLINT        NOT NULL DEFAULT 99  -- lower = prefer this warehouse
);

-- Inventory transfers between warehouses
CREATE TABLE inventory_transfers (
    transfer_id     UUID            PRIMARY KEY DEFAULT gen_random_uuid(),
    sku_id          VARCHAR(50)     NOT NULL,
    from_warehouse  SMALLINT        NOT NULL REFERENCES warehouses(warehouse_id),
    to_warehouse    SMALLINT        NOT NULL REFERENCES warehouses(warehouse_id),
    quantity        INTEGER         NOT NULL CHECK (quantity > 0),
    status          VARCHAR(20)     NOT NULL DEFAULT 'REQUESTED',
                                    -- REQUESTED, APPROVED, IN_TRANSIT, RECEIVED, CANCELLED
    requested_at    TIMESTAMPTZ     NOT NULL DEFAULT now(),
    shipped_at      TIMESTAMPTZ,
    received_at     TIMESTAMPTZ,
    notes           TEXT,
    requested_by    VARCHAR(100)    NOT NULL
);

-- Outbox for guaranteed event publishing
CREATE TABLE outbox_events (
    id              BIGSERIAL       PRIMARY KEY,
    topic           VARCHAR(100)    NOT NULL,
    partition_key   VARCHAR(100)    NOT NULL,
    payload         JSONB           NOT NULL,
    published       BOOLEAN         NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ     NOT NULL DEFAULT now()
);

CREATE INDEX idx_outbox_unpublished
    ON outbox_events (id)
    WHERE published = false;
```

### MongoDB — Audit and Reconciliation Store

```javascript
// inventory_snapshots — daily warehouse-level snapshots
{
  _id: ObjectId(),
  warehouseId: 5,
  snapshotDate: ISODate("2026-03-24"),
  snapshotType: "DAILY_EOD",     // or "PRE_RECONCILIATION"
  totalSkus: 100000,
  totalUnits: 2387456,
  items: [
    { skuId: "SKU-12345", quantityOnHand: 150, quantityOnHold: 12 },
    { skuId: "SKU-67890", quantityOnHand: 0,   quantityOnHold: 0  }
    // ... compressed per-SKU snapshot
  ],
  takenAt: ISODate("2026-03-24T23:59:59Z"),
  takenByJobId: "snapshot-job-20260324"
}

// reconciliation_sessions — tracks physical count sessions
{
  _id: ObjectId(),
  sessionId: "RECON-2026-03-24-WH05",
  warehouseId: 5,
  sessionDate: ISODate("2026-03-24"),
  status: "COMPLETED",  // IN_PROGRESS, COMPLETED, FAILED
  systemSnapshotId: "...",
  countsSubmitted: 100000,
  discrepanciesFound: 47,
  discrepanciesResolved: 45,
  startedAt: ISODate(),
  completedAt: ISODate(),
  performedBy: "warehouse-staff-team-b"
}

// discrepancies — items where physical count != system count
{
  _id: ObjectId(),
  sessionId: "RECON-2026-03-24-WH05",
  warehouseId: 5,
  skuId: "SKU-99887",
  systemQuantity: 100,
  physicalCount: 97,
  delta: -3,              // negative = shrinkage, positive = phantom stock
  deltaPercent: -3.0,
  severity: "LOW",        // LOW (<1%), MEDIUM (1-5%), HIGH (>5%)
  status: "RESOLVED",     // PENDING, INVESTIGATING, RESOLVED, ESCALATED
  resolutionType: "ADJUSTMENT",  // ADJUSTMENT, WRITE_OFF, RECOUNT, PENDING_AUDIT
  adjustmentApprovedBy: "manager-john",
  createdAt: ISODate(),
  resolvedAt: ISODate()
}
```

### Redis Key Structure

```
inv:level:{warehouseId}:{skuId}
  Type: Hash
  Fields:
    on_hand      → "150"
    on_hold      → "12"
    available    → "138"   (on_hand - on_hold, pre-computed)
    version      → "4823"  (mirrors PostgreSQL version col)
  TTL: none (permanent, warmed from DB)

inv:warehouse:{warehouseId}:meta
  Type: Hash
  Fields:
    name, city, state, zip, lat, lng, zone, active

inv:sku:warehouses:{skuId}
  Type: Sorted Set
  Score: available quantity (descending for ZREVRANGEBYSCOREsearch)
  Members: warehouseId strings
  Use: fast "which warehouses have stock for SKU X?" query
  TTL: 60 seconds (refreshed on every movement)

inv:hold:lock:{orderId}
  Type: String (distributed lock)
  TTL: 30 seconds

inv:reorder:alerts
  Type: Set
  Members: "warehouseId:skuId" pairs below reorder point
  Purpose: alert service polling

rate:inv:write:{warehouseId}
  Type: String (token bucket counter)
  TTL: 1 second
```

---

## Question 2: Handling Concurrent Inventory Updates

### Two-Layer Concurrency Control

**Layer 1 – Redis Lua for Speed (non-final)**
For inventory hold placement (order reservation), use atomic Lua scripts in Redis to check and decrement available stock with zero race conditions. This is the fast path — no DB write yet, just a Redis reservation.

**Layer 2 – PostgreSQL Optimistic Locking (final commit)**
When confirming or finalising a hold, write to PostgreSQL using optimistic locking (`version` column). If version mismatch → retry. Pessimistic locking (SELECT FOR UPDATE) is avoided except for transfers.

### Redis Lua Script — Atomic Hold Placement

```lua
-- KEYS[1] = inv:level:{warehouseId}:{skuId}
-- KEYS[2] = inv:sku:warehouses:{skuId}
-- ARGV[1] = quantity requested
-- ARGV[2] = warehouseId
-- ARGV[3] = holdTtlSeconds (e.g. 900 for 15 min)
-- Returns: {newAvailable, previousAvailable} or {-1, currentAvailable} if insufficient

local key    = KEYS[1]
local zset   = KEYS[2]
local qty    = tonumber(ARGV[1])
local wid    = ARGV[2]
local ttl    = tonumber(ARGV[3])

-- Read current state atomically
local onHand = tonumber(redis.call('HGET', key, 'on_hand') or '0')
local onHold = tonumber(redis.call('HGET', key, 'on_hold') or '0')
local available = onHand - onHold

if available < qty then
    return {-1, available}
end

-- Atomically increment hold
local newOnHold   = onHold + qty
local newAvailable = onHand - newOnHold

redis.call('HSET', key,
    'on_hold',  tostring(newOnHold),
    'available', tostring(newAvailable))

-- Update sorted set for warehouse selection queries
redis.call('ZADD', zset, newAvailable, wid)

-- Do NOT set TTL on inv:level (it's permanent)
-- Caller registers hold expiry separately

return {newAvailable, available}
```

### Spring Boot — Inventory Hold Service

```java
@Service
@Slf4j
public class InventoryHoldService {

    private static final String HOLD_LUA_SCRIPT = """
        local key    = KEYS[1]
        local zset   = KEYS[2]
        local qty    = tonumber(ARGV[1])
        local wid    = ARGV[2]
        -- ... (full script above)
        return {newAvailable, available}
        """;

    private final RedisTemplate<String, String> redis;
    private final DefaultRedisScript<List<Long>> holdScript;
    private final InventoryLevelRepository inventoryRepo;
    private final InventoryHoldRepository holdRepo;
    private final OutboxEventRepository outboxRepo;

    @PostConstruct
    public void init() {
        holdScript = new DefaultRedisScript<>();
        holdScript.setScriptText(HOLD_LUA_SCRIPT);
        holdScript.setResultType((Class<List<Long>>) (Class<?>) List.class);
    }

    /**
     * Place an inventory hold. Two-phase:
     *  1. Atomic Redis Lua reserve (fast path, ~1ms)
     *  2. PostgreSQL write with optimistic lock (durable, ~10ms)
     */
    @Transactional
    public InventoryHold placeHold(
            int warehouseId, String skuId, String orderId,
            int quantity, Duration ttl) {

        // Phase 1: Redis atomic reserve
        String redisKey  = "inv:level:" + warehouseId + ":" + skuId;
        String zsetKey   = "inv:sku:warehouses:" + skuId;

        List<Long> result = redis.execute(holdScript,
            List.of(redisKey, zsetKey),
            String.valueOf(quantity),
            String.valueOf(warehouseId),
            String.valueOf(ttl.toSeconds()));

        long newAvailable = result.get(0);
        if (newAvailable == -1) {
            throw new InsufficientStockException(
                warehouseId, skuId, quantity, result.get(1).intValue());
        }

        // Phase 2: PostgreSQL optimistic lock — persist the hold
        InventoryLevel level = inventoryRepo
            .findByWarehouseIdAndSkuId(warehouseId, skuId)
            .orElseThrow(() -> new StockRecordNotFoundException(warehouseId, skuId));

        // Optimistic lock: version check in UPDATE WHERE clause
        int updated = inventoryRepo.incrementHoldWithVersionCheck(
            warehouseId, skuId, quantity, level.getVersion());

        if (updated == 0) {
            // Concurrent update — roll back Redis optimistically
            rollbackRedisHold(redisKey, zsetKey, warehouseId, quantity);
            throw new ConcurrentInventoryUpdateException(warehouseId, skuId);
        }

        // Persist hold record + outbox event atomically
        Instant expiresAt = Instant.now().plus(ttl);
        InventoryHold hold = InventoryHold.builder()
            .warehouseId(warehouseId)
            .skuId(skuId)
            .orderId(orderId)
            .quantity(quantity)
            .holdStatus(HoldStatus.ACTIVE)
            .expiresAt(expiresAt)
            .build();

        holdRepo.save(hold);

        // Register expiry timer key in Redis (keyspace notification → auto-release)
        redis.opsForValue().set(
            "inv:hold:expiry:" + hold.getHoldId(),
            warehouseId + ":" + skuId + ":" + quantity,
            ttl);

        // Outbox event
        outboxRepo.save(OutboxEvent.builder()
            .topic("inventory.holds")
            .partitionKey(skuId)
            .payload(buildHoldPlacedEvent(hold))
            .build());

        log.info("Hold placed: orderId={} warehouse={} sku={} qty={} expires={}",
            orderId, warehouseId, skuId, quantity, expiresAt);

        return hold;
    }

    /** Confirm hold → permanent deduction (order confirmed) */
    @Transactional
    public void confirmHold(String holdId) {
        InventoryHold hold = holdRepo.findById(holdId)
            .filter(h -> h.getHoldStatus() == HoldStatus.ACTIVE)
            .orElseThrow(() -> new HoldNotFoundException(holdId));

        // Move on_hold → actual deduction (reduce on_hand AND on_hold)
        inventoryRepo.confirmHold(
            hold.getWarehouseId(), hold.getSkuId(), hold.getQuantity());

        // Update Redis: reduce on_hand (on_hold already reduced by hold placement logic)
        String redisKey = "inv:level:" + hold.getWarehouseId() + ":" + hold.getSkuId();
        redis.execute((RedisCallback<Void>) conn -> {
            conn.hIncrBy(redisKey.getBytes(), "on_hand".getBytes(), -hold.getQuantity());
            conn.hIncrBy(redisKey.getBytes(), "on_hold".getBytes(), -hold.getQuantity());
            // available stays the same (on_hand - on_hold unchanged)
            return null;
        });

        // Log movement
        recordMovement(hold, MovementType.SALE, -hold.getQuantity());

        hold.setHoldStatus(HoldStatus.CONFIRMED);
        hold.setConfirmedAt(Instant.now());
        holdRepo.save(hold);

        // Delete expiry key — no longer needed
        redis.delete("inv:hold:expiry:" + holdId);
    }

    /** Release hold → restore available (order cancelled / timed out) */
    @Transactional
    public void releaseHold(String holdId, String reason) {
        InventoryHold hold = holdRepo.findById(holdId)
            .filter(h -> h.getHoldStatus() == HoldStatus.ACTIVE)
            .orElseThrow(() -> new HoldNotFoundException(holdId));

        // Restore on_hold in PostgreSQL
        inventoryRepo.releaseHold(
            hold.getWarehouseId(), hold.getSkuId(), hold.getQuantity());

        // Restore available in Redis
        String redisKey = "inv:level:" + hold.getWarehouseId() + ":" + hold.getSkuId();
        String zsetKey  = "inv:sku:warehouses:" + hold.getSkuId();
        redis.execute((RedisCallback<Void>) conn -> {
            conn.hIncrBy(redisKey.getBytes(), "on_hold".getBytes(), -hold.getQuantity());
            conn.hIncrBy(redisKey.getBytes(), "available".getBytes(), hold.getQuantity());
            // Update sorted set score
            Long newAvail = Long.parseLong(new String(
                conn.hGet(redisKey.getBytes(), "available".getBytes())));
            conn.zAdd(zsetKey.getBytes(), newAvail,
                String.valueOf(hold.getWarehouseId()).getBytes());
            return null;
        });

        hold.setHoldStatus(HoldStatus.RELEASED);
        hold.setReleasedAt(Instant.now());
        holdRepo.save(hold);

        outboxRepo.save(OutboxEvent.builder()
            .topic("inventory.holds")
            .partitionKey(hold.getSkuId())
            .payload(buildHoldReleasedEvent(hold, reason))
            .build());
    }
}
```

### PostgreSQL — Optimistic Lock Update Query

```java
@Repository
public interface InventoryLevelRepository extends JpaRepository<InventoryLevel, Long> {

    // Optimistic locking via WHERE version = :version
    // Returns 1 if updated, 0 if version mismatch (concurrent update)
    @Modifying
    @Query("""
        UPDATE InventoryLevel i
        SET i.quantityOnHold = i.quantityOnHold + :qty,
            i.version = i.version + 1,
            i.lastMovementAt = CURRENT_TIMESTAMP
        WHERE i.warehouseId = :wid
          AND i.skuId = :skuId
          AND i.version = :expectedVersion
          AND (i.quantityOnHand - i.quantityOnHold) >= :qty
        """)
    int incrementHoldWithVersionCheck(
        @Param("wid")             int warehouseId,
        @Param("skuId")           String skuId,
        @Param("qty")             int qty,
        @Param("expectedVersion") long expectedVersion);

    @Modifying
    @Query("""
        UPDATE InventoryLevel i
        SET i.quantityOnHand = i.quantityOnHand - :qty,
            i.quantityOnHold = i.quantityOnHold - :qty,
            i.version = i.version + 1,
            i.lastMovementAt = CURRENT_TIMESTAMP
        WHERE i.warehouseId = :wid AND i.skuId = :skuId
          AND i.quantityOnHold >= :qty
        """)
    int confirmHold(
        @Param("wid")   int warehouseId,
        @Param("skuId") String skuId,
        @Param("qty")   int qty);

    @Modifying
    @Query("""
        UPDATE InventoryLevel i
        SET i.quantityOnHold = i.quantityOnHold - :qty,
            i.version = i.version + 1
        WHERE i.warehouseId = :wid AND i.skuId = :skuId
          AND i.quantityOnHold >= :qty
        """)
    int releaseHold(
        @Param("wid")   int warehouseId,
        @Param("skuId") String skuId,
        @Param("qty")   int qty);
}
```

### Hold Expiry — Redis Keyspace Notifications

```java
/**
 * Listens for Redis keyspace expiry events on inv:hold:expiry:* keys.
 * When a hold TTL expires, automatically releases the hold.
 */
@Component
@Slf4j
public class HoldExpiryListener implements MessageListener {

    private final InventoryHoldService holdService;

    @Override
    public void onMessage(Message message, byte[] pattern) {
        String expiredKey = new String(message.getBody());
        // expiredKey = "inv:hold:expiry:{holdId}"

        if (!expiredKey.startsWith("inv:hold:expiry:")) return;

        String holdId = expiredKey.replace("inv:hold:expiry:", "");
        log.info("Hold TTL expired, auto-releasing holdId={}", holdId);

        try {
            holdService.releaseHold(holdId, "TTL_EXPIRED");
        } catch (HoldNotFoundException e) {
            // Already confirmed or released — safe to ignore
            log.debug("Hold {} already resolved when expiry fired", holdId);
        }
    }
}

// Safety-net scheduled job (catches any keyspace notification misses)
@Scheduled(fixedDelay = 60_000)
@Transactional
public void sweepExpiredHolds() {
    List<InventoryHold> expired = holdRepo
        .findByHoldStatusAndExpiresAtBefore(HoldStatus.ACTIVE, Instant.now());

    for (InventoryHold hold : expired) {
        holdService.releaseHold(hold.getHoldId(), "SWEEP_EXPIRED");
    }

    if (!expired.isEmpty()) {
        log.warn("Sweep released {} expired holds (keyspace notification missed)",
            expired.size());
    }
}
```

---

## Question 3: Warehouse Selection Algorithm

### Goal
When an order is placed for SKU-X, quantity Q, shipping to ZIP-12345, find the **optimal warehouse** based on:
1. Has sufficient available stock
2. Shipping zone distance (lower = cheaper, faster)
3. Warehouse priority rank (fulfillment center vs regional)
4. Current load factor (don't overload one warehouse)
5. Transfer cost (avoid inter-warehouse stock imbalance)

### Warehouse Scoring Model

```
Score = α·ZoneScore + β·StockScore + γ·PriorityScore + δ·LoadScore

ZoneScore   = (maxZone - shippingZone) / maxZone        [0–1, higher = closer]
StockScore  = min(availableQty / requestedQty, 2) / 2   [0–1, capped at 2x requested]
PriorityScore = (maxRank - warehousePriorityRank) / maxRank  [0–1]
LoadScore   = 1 - (activeHolds / capacity)              [0–1, higher = less loaded]

Weights: α=0.40, β=0.35, γ=0.15, δ=0.10
```

### Shipping Zone Calculation

UPS/FedEx zones are computed from origin ZIP to destination ZIP.
Pre-computed and stored per warehouse for fast lookup.

```sql
-- Zones from each warehouse to each US state (simplified)
CREATE TABLE warehouse_shipping_zones (
    warehouse_id    SMALLINT    NOT NULL,
    destination_zip_prefix CHAR(3) NOT NULL,  -- first 3 digits
    zone            SMALLINT    NOT NULL,      -- 1=cheapest, 8=most expensive
    PRIMARY KEY (warehouse_id, destination_zip_prefix)
);
```

### Warehouse Router Service

```java
@Service
@Slf4j
public class WarehouseRouterService {

    private static final double ALPHA = 0.40;   // Zone weight
    private static final double BETA  = 0.35;   // Stock weight
    private static final double GAMMA = 0.15;   // Priority weight
    private static final double DELTA = 0.10;   // Load weight

    private final RedisTemplate<String, String> redis;
    private final WarehouseShippingZoneRepository zoneRepo;
    private final InventoryLevelRepository inventoryRepo;

    /**
     * Select the best warehouse to fulfil an order.
     * Fast path: Redis sorted set for stock check, zone from cache.
     * Slow path: DB query if Redis cache miss.
     */
    public WarehouseSelectionResult selectWarehouse(
            String skuId, int quantity, String destinationZip) {

        // Step 1: Find candidate warehouses with sufficient stock
        // Redis sorted set: inv:sku:warehouses:{skuId} scored by available qty
        Set<TypedTuple<String>> candidates = redis.opsForZSet()
            .reverseRangeByScoreWithScores("inv:sku:warehouses:" + skuId,
                quantity,      // min score = at least 'quantity' available
                Double.MAX_VALUE,
                0, 20);        // top 20 candidates max

        if (candidates == null || candidates.isEmpty()) {
            // Redis cache miss — fallback to DB
            candidates = loadCandidatesFromDb(skuId, quantity);
        }

        if (candidates.isEmpty()) {
            return WarehouseSelectionResult.outOfStock(skuId, quantity);
        }

        String zipPrefix = destinationZip.substring(0, 3);
        double maxAvailable = candidates.stream()
            .mapToDouble(t -> t.getScore())
            .max().orElse(1.0);

        // Step 2: Score each candidate
        List<WarehouseScore> scored = candidates.stream()
            .map(candidate -> {
                int warehouseId = Integer.parseInt(candidate.getValue());
                double available = candidate.getScore();

                // Get zone from cache
                int zone = getZone(warehouseId, zipPrefix);

                // Get warehouse metadata from Redis
                Map<Object, Object> meta = redis.opsForHash()
                    .entries("inv:warehouse:" + warehouseId + ":meta");
                int priority = Integer.parseInt(
                    (String) meta.getOrDefault("priority_rank", "99"));
                double loadFactor = Double.parseDouble(
                    (String) meta.getOrDefault("load_factor", "0.5"));

                // Compute score
                double zoneScore     = (8.0 - zone) / 8.0;
                double stockScore    = Math.min(available / quantity, 2.0) / 2.0;
                double priorityScore = (100.0 - priority) / 100.0;
                double loadScore     = 1.0 - loadFactor;

                double totalScore = ALPHA * zoneScore
                    + BETA  * stockScore
                    + GAMMA * priorityScore
                    + DELTA * loadScore;

                return new WarehouseScore(warehouseId, totalScore, zone,
                    (int) available, priority);
            })
            .sorted(Comparator.comparingDouble(WarehouseScore::score).reversed())
            .collect(Collectors.toList());

        WarehouseScore best = scored.get(0);
        log.debug("Warehouse selection: sku={} qty={} zip={} → warehouse={} score={}",
            skuId, quantity, destinationZip, best.warehouseId(), best.score());

        return WarehouseSelectionResult.selected(
            best.warehouseId(), best.zone(), best.availableQty(), scored);
    }

    /**
     * Multi-SKU order: select warehouse(s) that can fulfil ALL items,
     * preferring single-warehouse fulfilment to reduce split shipments.
     */
    public List<WarehouseAllocation> selectForMultiSkuOrder(
            List<OrderLineItem> items, String destinationZip) {

        // Step 1: Try single warehouse fulfilment (best for customer)
        Optional<Integer> singleWarehouse = trySingleWarehouseFulfilment(
            items, destinationZip);

        if (singleWarehouse.isPresent()) {
            return items.stream()
                .map(item -> new WarehouseAllocation(
                    singleWarehouse.get(), item.skuId(), item.quantity()))
                .collect(Collectors.toList());
        }

        // Step 2: Multi-warehouse split (minimise number of splits)
        return splitFulfilment(items, destinationZip);
    }

    private Optional<Integer> trySingleWarehouseFulfilment(
            List<OrderLineItem> items, String destinationZip) {

        // Find warehouses with stock for ALL line items
        Set<Integer> commonWarehouses = null;
        for (OrderLineItem item : items) {
            Set<Integer> withStock = getWarehousesWithStock(
                item.skuId(), item.quantity());

            if (commonWarehouses == null) {
                commonWarehouses = withStock;
            } else {
                commonWarehouses.retainAll(withStock);
            }
            if (commonWarehouses.isEmpty()) break;
        }

        if (commonWarehouses == null || commonWarehouses.isEmpty()) {
            return Optional.empty();
        }

        // From warehouses that can fulfil all items, pick best by zone
        String zipPrefix = destinationZip.substring(0, 3);
        return commonWarehouses.stream()
            .min(Comparator.comparingInt(wid -> getZone(wid, zipPrefix)));
    }

    private int getZone(int warehouseId, String zipPrefix) {
        // L1: Redis hash cache
        String zoneKey = "inv:zone:" + warehouseId + ":" + zipPrefix;
        String cached = redis.opsForValue().get(zoneKey);
        if (cached != null) return Integer.parseInt(cached);

        // L2: DB lookup
        int zone = zoneRepo.findZone(warehouseId, zipPrefix)
            .orElse(8);  // default to most expensive zone if unknown
        redis.opsForValue().set(zoneKey, String.valueOf(zone), Duration.ofHours(24));
        return zone;
    }
}
```

---

## Question 4: Inventory Holds vs Available Inventory

### Three-Level Inventory Model

```
┌─────────────────────────────────────────────────────────────┐
│               quantity_on_hand  (physical stock in warehouse)│
│                                                              │
│  ┌────────────────────┐    ┌─────────────────────────────┐  │
│  │  quantity_on_hold  │    │  quantity_available          │  │
│  │  (reserved for     │    │  = on_hand - on_hold         │  │
│  │  pending orders)   │    │  (can be sold)               │  │
│  └────────────────────┘    └─────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘

+ quantity_in_transit: stock en route from another warehouse
  (already committed from source, not yet received at destination)
```

### Hold Lifecycle State Machine

```
                 placeHold()
NONE ───────────────────────────► ACTIVE (15-min TTL)
                                      │
                          ┌───────────┼────────────┐
                          │           │            │
                  confirmHold()  releaseHold()  TTL_EXPIRES
                          │           │     (keyspace notification)
                          ▼           ▼            ▼
                      CONFIRMED   RELEASED      RELEASED
                       (stock          (stock       (stock
                       deducted)       restored)    restored)
```

### Stock Visibility Rules

| Operation | on_hand | on_hold | available | in_transit |
|---|---|---|---|---|
| Place hold (order placed) | unchanged | +qty | -qty | unchanged |
| Confirm hold (order paid) | -qty | -qty | unchanged | unchanged |
| Release hold (order cancelled) | unchanged | -qty | +qty | unchanged |
| Sale (no hold) | -qty | unchanged | -qty | unchanged |
| Restock (PO received) | +qty | unchanged | +qty | unchanged |
| Transfer out | -qty | unchanged | -qty | +qty (destination) |
| Transfer in | +qty | unchanged | +qty | -qty |
| Return | +qty | unchanged | +qty | unchanged |

### Available Stock Computation

```java
@Service
public class InventoryQueryService {

    private final RedisTemplate<String, String> redis;
    private final InventoryLevelRepository inventoryRepo;

    /** Real-time available stock for a SKU at a specific warehouse */
    public int getAvailableStock(int warehouseId, String skuId) {
        // Always serve from Redis (sub-millisecond)
        Map<Object, Object> data = redis.opsForHash()
            .entries("inv:level:" + warehouseId + ":" + skuId);

        if (!data.isEmpty()) {
            return Integer.parseInt((String) data.getOrDefault("available", "0"));
        }

        // Redis miss → load from DB and warm cache
        return loadAndCacheFromDb(warehouseId, skuId);
    }

    /**
     * Batch available stock query (e.g., for product catalog display).
     * Uses Redis pipelining for efficiency.
     */
    public Map<String, Integer> getAvailableStockBatch(
            List<String> skuIds, int warehouseId) {

        List<Object> results = redis.executePipelined((RedisCallback<Object>) conn -> {
            for (String skuId : skuIds) {
                String key = "inv:level:" + warehouseId + ":" + skuId;
                conn.hashCommands().hGet(key.getBytes(), "available".getBytes());
            }
            return null;
        });

        Map<String, Integer> stockMap = new HashMap<>();
        for (int i = 0; i < skuIds.size(); i++) {
            byte[] val = (byte[]) results.get(i);
            int available = val != null ? Integer.parseInt(new String(val)) : 0;
            stockMap.put(skuIds.get(i), available);
        }
        return stockMap;
    }

    /**
     * Network-level view: available across ALL warehouses for a SKU.
     * Uses the sorted set for fast result.
     */
    public List<WarehouseStock> getNetworkAvailability(String skuId) {
        // Redis sorted set ranked by available quantity
        Set<TypedTuple<String>> warehouseStocks = redis.opsForZSet()
            .reverseRangeWithScores("inv:sku:warehouses:" + skuId, 0, -1);

        return warehouseStocks.stream()
            .map(t -> new WarehouseStock(
                Integer.parseInt(t.getValue()),
                t.getScore().intValue()))
            .filter(ws -> ws.available() > 0)
            .collect(Collectors.toList());
    }
}
```

---

## Question 5: Inventory Discrepancies During Reconciliation

### Reconciliation Flow

Daily physical counts are performed at each warehouse. The process compares physical counts against the system snapshot taken at count start.

```
T=00:00 (midnight)         T=02:00               T=06:00
    │                           │                     │
    ▼                           ▼                     ▼
Take System               Staff submit         Reconciliation
Snapshot                  physical counts      engine compares
(MongoDB)                 (via mobile app)     counts vs snapshot
                                                ↓
                                        Classify discrepancies
                                        AUTO-ADJUST < threshold
                                        FLAG > threshold for review
```

### Reconciliation Service

```java
@Service
@Slf4j
public class ReconciliationService {

    // Auto-approve adjustments within ±5 units OR ±1% of quantity, whichever is smaller
    private static final int    AUTO_APPROVE_UNIT_THRESHOLD    = 5;
    private static final double AUTO_APPROVE_PERCENT_THRESHOLD = 0.01;

    private final InventoryLevelRepository inventoryRepo;
    private final ReconciliationSessionRepository sessionRepo;
    private final DiscrepancyRepository discrepancyRepo;
    private final InventoryMovementRepository movementRepo;
    private final OutboxEventRepository outboxRepo;
    private final MongoTemplate mongoTemplate;
    private final RedisTemplate<String, String> redis;

    /**
     * Process physical count submissions for a warehouse.
     * Compares to system snapshot taken at session start.
     */
    @Transactional
    public ReconciliationResult processCountSubmissions(
            String sessionId,
            List<PhysicalCountSubmission> submissions) {

        ReconciliationSession session = sessionRepo.findById(sessionId)
            .orElseThrow(() -> new SessionNotFoundException(sessionId));

        // Load the pre-count system snapshot
        InventorySnapshot snapshot = loadSnapshot(session.getSystemSnapshotId());

        List<Discrepancy> discrepancies = new ArrayList<>();
        List<AutoApprovedAdjustment> autoApproved = new ArrayList<>();
        List<Discrepancy> needsReview = new ArrayList<>();

        for (PhysicalCountSubmission sub : submissions) {
            Integer systemQty = snapshot.getQuantityForSku(sub.skuId());
            if (systemQty == null) systemQty = 0;

            int delta = sub.physicalCount() - systemQty;
            if (delta == 0) continue;  // No discrepancy

            double deltaPercent = systemQty > 0
                ? Math.abs((double) delta / systemQty)
                : 1.0;

            Severity severity = classifySeverity(deltaPercent);

            Discrepancy disc = Discrepancy.builder()
                .sessionId(sessionId)
                .warehouseId(session.getWarehouseId())
                .skuId(sub.skuId())
                .systemQuantity(systemQty)
                .physicalCount(sub.physicalCount())
                .delta(delta)
                .deltaPercent(deltaPercent * 100)
                .severity(severity)
                .status(DiscrepancyStatus.PENDING)
                .build();

            discrepancies.add(disc);

            // Auto-approve small discrepancies
            boolean withinUnitThreshold = Math.abs(delta) <= AUTO_APPROVE_UNIT_THRESHOLD;
            boolean withinPctThreshold  = deltaPercent <= AUTO_APPROVE_PERCENT_THRESHOLD;

            if (withinUnitThreshold || withinPctThreshold) {
                disc.setStatus(DiscrepancyStatus.RESOLVED);
                disc.setResolutionType(ResolutionType.AUTO_ADJUSTMENT);
                autoApproved.add(new AutoApprovedAdjustment(disc));
            } else {
                needsReview.add(disc);
            }
        }

        // Persist all discrepancies to MongoDB
        mongoTemplate.insertAll(discrepancies);

        // Apply auto-approved adjustments atomically
        if (!autoApproved.isEmpty()) {
            applyBulkAdjustments(session.getWarehouseId(), autoApproved);
        }

        // Update session status
        session.setDiscrepanciesFound(discrepancies.size());
        session.setDiscrepanciesResolved(autoApproved.size());
        session.setStatus(needsReview.isEmpty()
            ? SessionStatus.COMPLETED
            : SessionStatus.PENDING_REVIEW);
        sessionRepo.save(session);

        // Publish for alerting
        outboxRepo.save(OutboxEvent.builder()
            .topic("inventory.reconcile")
            .partitionKey(String.valueOf(session.getWarehouseId()))
            .payload(buildReconciliationCompletedEvent(session, discrepancies))
            .build());

        log.info("Reconciliation session={}: {} discrepancies, {} auto-adjusted, {} pending review",
            sessionId, discrepancies.size(), autoApproved.size(), needsReview.size());

        return new ReconciliationResult(session, autoApproved, needsReview);
    }

    /**
     * Manager approves or rejects a flagged discrepancy.
     */
    @Transactional
    public void resolveDiscrepancy(
            String discrepancyId, ResolutionDecision decision,
            String approvedBy, String notes) {

        Discrepancy disc = discrepancyRepo.findById(discrepancyId)
            .orElseThrow(() -> new DiscrepancyNotFoundException(discrepancyId));

        if (decision == ResolutionDecision.APPROVE_ADJUSTMENT) {
            // Apply the adjustment: bring system count to match physical
            applySingleAdjustment(disc.getWarehouseId(), disc.getSkuId(),
                disc.getDelta(), approvedBy, notes, disc.getSessionId());

            disc.setStatus(DiscrepancyStatus.RESOLVED);
            disc.setResolutionType(ResolutionType.MANAGER_ADJUSTMENT);
        } else if (decision == ResolutionDecision.RECOUNT) {
            disc.setStatus(DiscrepancyStatus.PENDING);
            disc.setResolutionType(ResolutionType.RECOUNT_REQUESTED);
        } else if (decision == ResolutionDecision.WRITE_OFF) {
            disc.setStatus(DiscrepancyStatus.RESOLVED);
            disc.setResolutionType(ResolutionType.WRITE_OFF);
            // Write-off: accept loss without physical adjustment
        }

        disc.setApprovedBy(approvedBy);
        disc.setNotes(notes);
        disc.setResolvedAt(Instant.now());
        discrepancyRepo.save(disc);
    }

    @Transactional
    private void applySingleAdjustment(
            int warehouseId, String skuId, int delta,
            String approvedBy, String notes, String referenceId) {

        // Update PostgreSQL atomically
        if (delta > 0) {
            inventoryRepo.increaseOnHand(warehouseId, skuId, delta);
        } else {
            // Ensure we don't go negative — use MAX(on_hand + delta, 0)
            inventoryRepo.decreaseOnHand(warehouseId, skuId, Math.abs(delta));
        }

        // Sync Redis
        String redisKey = "inv:level:" + warehouseId + ":" + skuId;
        redis.opsForHash().increment(redisKey, "on_hand", delta);
        redis.opsForHash().increment(redisKey, "available", delta);

        // Immutable movement record
        recordMovement(warehouseId, skuId, MovementType.CYCLE_COUNT_ADJUSTMENT,
            delta, approvedBy, notes, referenceId);
    }

    private Severity classifySeverity(double deltaPercent) {
        if (deltaPercent < 0.01) return Severity.LOW;
        if (deltaPercent < 0.05) return Severity.MEDIUM;
        return Severity.HIGH;
    }
}
```

---

## Question 6: Inventory Accuracy at High Transaction Volumes

### Challenge
At 50 warehouses × 100,000 SKUs, with concurrent sales, restocks, and returns, we need:
- **Zero oversells** — never confirm a hold that doesn't exist
- **Sub-5ms hold placement** — on the critical order checkout path
- **Eventual consistency** is acceptable for display (product pages), but **not** for holds/deductions

### Six-Layer Accuracy Strategy

**Layer 1 – Redis Lua (Atomic Fast Path)**
All hold placements go through a single Redis Lua script. Lua scripts are executed atomically (single-threaded Redis) — no two concurrent requests can interleave. This is the first line of defence.

**Layer 2 – PostgreSQL Optimistic Locking**
Every write to `inventory_levels` uses `WHERE version = :expectedVersion`. Any concurrent update increments `version`, causing the second writer to get `0 rows updated` → caught and retried.

**Layer 3 – Database CHECK Constraints**
```sql
CHECK (quantity_on_hand >= 0)
CHECK (quantity_on_hold >= 0)
CHECK (quantity_on_hand >= quantity_on_hold)
```
These are the last line of defence — a bug can never produce negative stock in PostgreSQL.

**Layer 4 – Immutable Movement Log**
Every stock change writes an `inventory_movements` record in the same transaction. The running total can be verified at any time:
```sql
SELECT SUM(quantity_delta) as computed_balance
FROM inventory_movements
WHERE warehouse_id = 5 AND sku_id = 'SKU-123'
```
Must equal `quantity_on_hand`. Automated hourly verification job catches drift.

**Layer 5 – Redis/DB Consistency Watchdog**
Scheduled job runs every 5 minutes, sampling random 1,000 warehouse+SKU pairs, comparing Redis `available` to PostgreSQL computed `(on_hand - on_hold)`. Any mismatch → Redis refreshed from DB + alert fired.

**Layer 6 – Idempotency on All Writes**
Every inventory movement carries a `reference_id` (orderId, transferId, etc.). Unique constraint on `(warehouse_id, sku_id, reference_id, movement_type)` ensures duplicate Kafka events do not cause double-deductions.

```sql
CREATE UNIQUE INDEX idx_movements_idempotency
    ON inventory_movements (warehouse_id, sku_id, reference_id, movement_type)
    WHERE reference_id IS NOT NULL;
```

### Redis–PostgreSQL Sync Watchdog

```java
@Scheduled(fixedDelay = 300_000)  // every 5 minutes
@Slf4j
public void validateRedisDbConsistency() {
    // Sample 1,000 random warehouse+SKU pairs
    List<InventoryLevel> sample = inventoryRepo.findRandomSample(1000);

    int mismatches = 0;
    for (InventoryLevel level : sample) {
        String redisKey = "inv:level:" + level.getWarehouseId() + ":" + level.getSkuId();
        String redisAvail = (String) redis.opsForHash().get(redisKey, "available");

        if (redisAvail == null) continue;  // Not cached — skip

        int dbAvailable = level.getQuantityOnHand() - level.getQuantityOnHold();
        int redisAvailable = Integer.parseInt(redisAvail);

        if (dbAvailable != redisAvailable) {
            mismatches++;
            log.warn("CACHE DRIFT: warehouse={} sku={} db={} redis={}",
                level.getWarehouseId(), level.getSkuId(), dbAvailable, redisAvailable);

            // Self-heal: refresh from DB
            redis.opsForHash().put(redisKey, "available", String.valueOf(dbAvailable));
            redis.opsForHash().put(redisKey, "on_hand",
                String.valueOf(level.getQuantityOnHand()));
            redis.opsForHash().put(redisKey, "on_hold",
                String.valueOf(level.getQuantityOnHold()));

            meterRegistry.counter("inventory.cache.drift",
                "warehouse", String.valueOf(level.getWarehouseId()))
                .increment();
        }
    }

    if (mismatches > 10) {
        alertService.fire(AlertLevel.HIGH,
            "Inventory cache drift: " + mismatches + " mismatches in 1000-sample");
    }
}
```

---

## Inventory Transfer Workflow

Transfers are handled as a Saga: subtract from source → confirm in-transit → add to destination.

```
Source Warehouse (WH-A)          Transfer Service         Destination (WH-B)
        │                               │                        │
        │ ←── TRANSFER_OUT event ───────│                        │
        │    Lua: decrease available    │                        │
        │    PG: decrease on_hand       │                        │
        │    PG: record movement        │                        │
        │                               │                        │
        │                               │──── in_transit +qty ──→│
        │                               │    PG: record pending  │
        │                               │    transfer record     │
        │                               │                        │
        │                               │ ←── TRANSFER_RECEIVED ─│
        │                               │    (warehouse confirms  │
        │                               │     goods arrived)     │
        │                               │                        │
        │                               │──── TRANSFER_COMPLETE ─│
        │                               │    PG: on_hand + qty   │
        │                               │    Redis: available +qty│
        │                               │    record movement     │
```

```java
@Service
@Slf4j
public class TransferService {

    @Transactional
    public InventoryTransfer initiateTransfer(
            String skuId, int fromWarehouseId, int toWarehouseId,
            int quantity, String requestedBy) {

        // Step 1: Check source has enough stock
        InventoryLevel source = inventoryRepo
            .findByWarehouseIdAndSkuId(fromWarehouseId, skuId)
            .orElseThrow(() -> new StockRecordNotFoundException(fromWarehouseId, skuId));

        if (source.getAvailableQuantity() < quantity) {
            throw new InsufficientStockException(fromWarehouseId, skuId, quantity,
                source.getAvailableQuantity());
        }

        // Step 2: Deduct from source atomically (optimistic lock)
        int updated = inventoryRepo.decreaseOnHandWithCheck(
            fromWarehouseId, skuId, quantity, source.getVersion());
        if (updated == 0) {
            throw new ConcurrentInventoryUpdateException(fromWarehouseId, skuId);
        }

        // Also deduct from Redis
        String srcKey = "inv:level:" + fromWarehouseId + ":" + skuId;
        redis.opsForHash().increment(srcKey, "on_hand", -quantity);
        redis.opsForHash().increment(srcKey, "available", -quantity);

        // Step 3: Mark destination as in-transit
        inventoryRepo.increaseInTransit(toWarehouseId, skuId, quantity);

        // Step 4: Create transfer record
        InventoryTransfer transfer = InventoryTransfer.builder()
            .skuId(skuId)
            .fromWarehouse(fromWarehouseId)
            .toWarehouse(toWarehouseId)
            .quantity(quantity)
            .status(TransferStatus.IN_TRANSIT)
            .requestedBy(requestedBy)
            .build();
        transferRepo.save(transfer);

        // Step 5: Record movements (outbox)
        recordMovement(fromWarehouseId, skuId, MovementType.TRANSFER_OUT,
            -quantity, requestedBy, null, transfer.getTransferId());

        outboxRepo.save(OutboxEvent.builder()
            .topic("inventory.transfers")
            .partitionKey(transfer.getTransferId())
            .payload(buildTransferInitiatedEvent(transfer))
            .build());

        return transfer;
    }

    @Transactional
    public void receiveTransfer(String transferId, int actualQuantityReceived) {
        InventoryTransfer transfer = transferRepo.findById(transferId)
            .filter(t -> t.getStatus() == TransferStatus.IN_TRANSIT)
            .orElseThrow(() -> new TransferNotFoundException(transferId));

        // Reduce in_transit at destination
        inventoryRepo.decreaseInTransit(
            transfer.getToWarehouse(), transfer.getSkuId(),
            transfer.getQuantity());

        // Add to on_hand at destination (may differ from sent if damaged in transit)
        inventoryRepo.increaseOnHand(
            transfer.getToWarehouse(), transfer.getSkuId(), actualQuantityReceived);

        // Update Redis destination
        String dstKey = "inv:level:" + transfer.getToWarehouse() + ":" + transfer.getSkuId();
        redis.opsForHash().increment(dstKey, "on_hand", actualQuantityReceived);
        redis.opsForHash().increment(dstKey, "available", actualQuantityReceived);

        // Handle short receipt (damaged/lost in transit)
        if (actualQuantityReceived < transfer.getQuantity()) {
            int lost = transfer.getQuantity() - actualQuantityReceived;
            log.warn("Transfer short receipt: transferId={} sent={} received={} lost={}",
                transferId, transfer.getQuantity(), actualQuantityReceived, lost);
            // Record transit loss movement at source for audit
            recordMovement(transfer.getFromWarehouse(), transfer.getSkuId(),
                MovementType.TRANSIT_LOSS, -lost, "SYSTEM",
                "Short receipt on transfer " + transferId, transferId);
        }

        recordMovement(transfer.getToWarehouse(), transfer.getSkuId(),
            MovementType.TRANSFER_IN, actualQuantityReceived, "SYSTEM",
            null, transferId);

        transfer.setStatus(TransferStatus.RECEIVED);
        transfer.setReceivedAt(Instant.now());
        transferRepo.save(transfer);
    }
}
```

---

## Kafka Topics and Event Flow

```
Topic: inventory.movements
  Partition key: skuId (guarantees ordering per SKU across warehouses)
  Events:
    InventoryDeducted      { warehouseId, skuId, orderId, qty, quantityAfter }
    InventoryRestocked     { warehouseId, skuId, poId, qty, quantityAfter }
    InventoryReturned      { warehouseId, skuId, orderId, qty, quantityAfter }
    InventoryAdjusted      { warehouseId, skuId, delta, approvedBy, sessionId }

Topic: inventory.holds
  Partition key: orderId
  Events:
    HoldPlaced     { holdId, warehouseId, skuId, orderId, qty, expiresAt }
    HoldConfirmed  { holdId, orderId }
    HoldReleased   { holdId, orderId, reason }
    HoldExpired    { holdId, orderId }

Topic: inventory.transfers
  Partition key: transferId
  Events:
    TransferInitiated  { transferId, skuId, from, to, qty }
    TransferReceived   { transferId, actualQty }
    TransferCancelled  { transferId, reason }

Topic: inventory.alerts
  Partition key: warehouseId
  Events:
    LowStockAlert      { warehouseId, skuId, currentQty, reorderPoint }
    StockOutAlert      { warehouseId, skuId }
    ReorderTriggered   { warehouseId, skuId, reorderQty }

Topic: inventory.reconcile
  Partition key: warehouseId
  Events:
    ReconciliationStarted   { sessionId, warehouseId }
    ReconciliationCompleted { sessionId, discrepancyCount, autoAdjusted }
    DiscrepancyEscalated    { discrepancyId, severity, warehouseId, skuId }
```

### Low-Stock Alert Processor (Kafka Consumer)

```java
@Component
@Slf4j
public class InventoryMovementConsumer {

    @KafkaListener(topics = "inventory.movements",
                   groupId = "low-stock-alert-processor",
                   concurrency = "10")
    public void onInventoryMovement(InventoryMovedEvent event) {
        // Check if new quantity is at or below reorder point
        InventoryLevel level = inventoryRepo
            .findByWarehouseIdAndSkuId(event.getWarehouseId(), event.getSkuId())
            .orElse(null);

        if (level == null) return;

        if (event.getQuantityAfter() == 0) {
            alertProducer.sendAlert(AlertType.STOCK_OUT,
                event.getWarehouseId(), event.getSkuId(), 0);
        } else if (event.getQuantityAfter() <= level.getReorderPoint()) {
            alertProducer.sendAlert(AlertType.LOW_STOCK,
                event.getWarehouseId(), event.getSkuId(), event.getQuantityAfter());

            // Trigger automated reorder if configured
            if (level.getReorderQuantity() > 0) {
                purchaseOrderService.triggerReorder(
                    event.getWarehouseId(), event.getSkuId(),
                    level.getReorderQuantity());
            }
        }
    }
}
```

---

## Cache Warming Strategy

At startup and after any cache miss, Redis is warmed from PostgreSQL.

```java
@Component
@Slf4j
public class InventoryCacheWarmer implements ApplicationRunner {

    @Override
    public void run(ApplicationArguments args) {
        log.info("Starting inventory cache warm-up...");
        Instant start = Instant.now();

        // Warm all 50 warehouses × 100K SKUs in batches
        // Use JDBC batch read → Redis pipeline batch write
        int batchSize = 1000;
        int totalLoaded = 0;

        try (Stream<InventoryLevel> stream = inventoryRepo.streamAll()) {
            List<InventoryLevel> batch = new ArrayList<>(batchSize);

            stream.forEach(level -> {
                batch.add(level);
                if (batch.size() >= batchSize) {
                    warmBatch(batch);
                    batch.clear();
                }
            });
            if (!batch.isEmpty()) warmBatch(batch);
        }

        log.info("Cache warm-up complete: {}ms",
            Duration.between(start, Instant.now()).toMillis());
    }

    private void warmBatch(List<InventoryLevel> levels) {
        redis.executePipelined((RedisCallback<Object>) conn -> {
            for (InventoryLevel level : levels) {
                String key = "inv:level:" + level.getWarehouseId() +
                    ":" + level.getSkuId();
                int available = level.getQuantityOnHand() - level.getQuantityOnHold();

                Map<byte[], byte[]> fields = Map.of(
                    "on_hand".getBytes(),  String.valueOf(level.getQuantityOnHand()).getBytes(),
                    "on_hold".getBytes(),  String.valueOf(level.getQuantityOnHold()).getBytes(),
                    "available".getBytes(), String.valueOf(available).getBytes(),
                    "version".getBytes(),  String.valueOf(level.getVersion()).getBytes()
                );
                conn.hashCommands().hMSet(key.getBytes(), fields);

                // Add to sorted set for warehouse selection
                String zsetKey = "inv:sku:warehouses:" + level.getSkuId();
                conn.zSetCommands().zAdd(zsetKey.getBytes(), available,
                    String.valueOf(level.getWarehouseId()).getBytes());
            }
            return null;
        });
    }
}
```

---

## Outbox Relay

```java
@Component
@Slf4j
public class OutboxRelay {

    private final OutboxEventRepository outboxRepo;
    private final KafkaTemplate<String, String> kafka;

    @Scheduled(fixedDelay = 100)
    @Transactional
    public void relayPendingEvents() {
        List<OutboxEvent> pending =
            outboxRepo.findTop100ByPublishedFalseOrderByIdAsc();

        for (OutboxEvent event : pending) {
            try {
                kafka.send(event.getTopic(), event.getPartitionKey(), event.getPayload())
                    .get(5, TimeUnit.SECONDS);
                event.setPublished(true);
                outboxRepo.save(event);
            } catch (Exception e) {
                log.error("Outbox relay failed for event id={}: {}",
                    event.getId(), e.getMessage());
                // Leave as unpublished — next poll will retry
            }
        }
    }
}
```

---

## Failure Scenarios and Mitigations

### Scenario 1: Redis Crash During High Traffic
```
Problem:
  Redis pod crashes.
  All hold placement requests fail (can't read available stock).

Mitigation:
  - Fallback to PostgreSQL direct query for available stock
    (add @Retry with Redis-unavailable exception class)
  - Redis cluster with Sentinel: auto-failover in ~30s
  - Caffeine L1 in-process cache (1-minute TTL) absorbs
    requests during Redis recovery window
  - Circuit breaker on Redis: open after 5 consecutive failures,
    route to DB fallback for 30s
```

### Scenario 2: Hold Placed in Redis but PostgreSQL Write Fails
```
Problem:
  Redis Lua script decrements available (success).
  PostgreSQL optimistic lock write fails (version mismatch).

Mitigation:
  - Detected immediately: incremented == 0
  - rollbackRedisHold() called: Lua script restores Redis state atomically
  - Request retried with fresh version read from DB
  - Max 3 retries with exponential backoff (50ms, 100ms, 200ms)
  - After 3 failures → 409 Conflict returned to caller
```

### Scenario 3: Hold TTL Key Lost (Redis Eviction / Restart)
```
Problem:
  inv:hold:expiry:{holdId} key evicted or lost on Redis restart.
  Hold never auto-released → stock permanently locked.

Mitigation:
  - Scheduled sweep job every 60 seconds queries DB for
    ACTIVE holds past their expires_at timestamp
  - Releases any found → safe self-healing
  - Dual safety: Redis keyspace notification (primary) +
    DB sweep (secondary)
```

### Scenario 4: Reconciliation Count Submitted Twice
```
Problem:
  Warehouse staff submits count twice (network retry).
  Adjustments applied twice → inventory corrupted.

Mitigation:
  - Reconciliation session has status: IN_PROGRESS → COMPLETED
  - Duplicate submission attempt returns 409 ALREADY_SUBMITTED
  - Movement idempotency key on CYCLE_COUNT_ADJUSTMENT:
    UNIQUE (warehouse_id, sku_id, reference_id='sessionId', movement_type='CYCLE_COUNT')
    → DB constraint rejects duplicate adjustment
```

### Scenario 5: Split-Brain During Warehouse Selection
```
Problem:
  Two concurrent orders both see available=1 for the last unit.
  Both pass warehouse selection, both place holds.

Mitigation:
  - Redis Lua script is atomic: only the first hold succeeds.
  - Second hold returns -1 (insufficient stock).
  - Second order must find an alternate warehouse or
    show "out of stock" to customer.
  - Result: zero oversells guaranteed by atomic Lua.
```

---

## Scaling Strategy

### Current Scale: 50 warehouses, 100K SKUs, ~1M inventory events/day

**PostgreSQL:**
- Table partitioning by `warehouse_id` (50 partitions) → each partition ~2M rows max
- Separate read replica for queries; primary for writes only
- Index on `(warehouse_id, sku_id)` handles all point lookups in O(log n)
- Outbox table with partial index on `published=false` → only scans ~100 unprocessed rows

**Redis:**
- Total keys: ~5M (50 warehouses × 100K SKUs)
- ~5M Hash keys + ~100K Sorted Sets (one per SKU)
- Memory estimate: ~1 GB for all inventory state (well within Redis capacity)
- Redis Cluster with 6 nodes for HA + horizontal partitioning

**Kafka:**
- `inventory.movements` with 20 partitions (keyed by skuId)
- Throughput: ~12 events/second average, spikes to ~5,000/sec during promotions

### Scaling to 500 Warehouses / 1M SKUs

If scale grows 10x:
- Shard PostgreSQL by `warehouse_id` across 5 clusters (10 warehouses per cluster)
- Redis memory grows to ~10 GB → still manageable, or shard Redis by `warehouseId % numShards`
- Kafka partitions: increase to 100 for `inventory.movements`
- Read-side: Elasticsearch index of network-level stock (`sku → [warehouses with stock]`) for product catalog

---

## REST API Design

```
# Inventory Queries (served from Redis via inventory-query-service)
GET /api/v1/inventory/{warehouseId}/{skuId}
    → { onHand, onHold, available, inTransit, version }

GET /api/v1/inventory/network/{skuId}
    → [ { warehouseId, available } ] sorted desc by available

GET /api/v1/inventory/{warehouseId}/batch?skus=SKU1,SKU2,...SKU500
    → { SKU1: available, SKU2: available, ... }

# Inventory Holds (served by inventory-service)
POST /api/v1/inventory/holds
    { warehouseId, skuId, orderId, quantity, ttlSeconds }
    → { holdId, expiresAt, newAvailable }

DELETE /api/v1/inventory/holds/{holdId}     → release hold
PUT    /api/v1/inventory/holds/{holdId}/confirm → confirm hold

# Warehouse Selection (served by warehouse-router-service)
POST /api/v1/warehouses/select
    { skuId, quantity, destinationZip }
    → { warehouseId, zone, score, alternatives: [...] }

POST /api/v1/warehouses/select/multi-sku
    { items: [{skuId, qty}], destinationZip }
    → [ { warehouseId, skuId, quantity } ]

# Transfers
POST /api/v1/transfers
    { skuId, fromWarehouseId, toWarehouseId, quantity }
    → { transferId, status }

POST /api/v1/transfers/{transferId}/receive
    { actualQuantityReceived }
    → { transferId, status: RECEIVED }

# Reconciliation
POST /api/v1/reconciliation/sessions
    { warehouseId }
    → { sessionId, systemSnapshotId }

POST /api/v1/reconciliation/sessions/{sessionId}/counts
    { counts: [ { skuId, physicalCount } ] }
    → { discrepanciesFound, autoAdjusted, pendingReview }

PUT /api/v1/reconciliation/discrepancies/{discrepancyId}/resolve
    { decision: APPROVE_ADJUSTMENT|RECOUNT|WRITE_OFF, approvedBy, notes }
```

---

## Monitoring and Alerting

### Key Metrics (Micrometer → Prometheus → Grafana)

```java
// Hold placement rate
Counter holdPlacedCounter = meterRegistry.counter("inventory.holds.placed",
    "warehouse", warehouseId);

// Hold failure rate (insufficient stock)
Counter holdFailedCounter = meterRegistry.counter("inventory.holds.failed",
    "warehouse", warehouseId, "reason", "INSUFFICIENT_STOCK");

// Cache drift detections
Counter cacheDriftCounter = meterRegistry.counter("inventory.cache.drift",
    "warehouse", warehouseId);

// Redis vs DB sync latency (Outbox relay lag)
Timer outboxRelayTimer = meterRegistry.timer("inventory.outbox.relay.latency");

// Reconciliation discrepancy rate per warehouse
Gauge discrepancyRate = meterRegistry.gauge("inventory.reconciliation.discrepancy_rate",
    () -> discrepancyRepo.getDiscrepancyRateForToday());
```

### Alert Rules

```yaml
# PagerDuty / Alertmanager rules
alerts:
  - name: InventoryStockOut
    condition: inventory_available == 0 AND is_active_sku == true
    severity: HIGH
    message: "Stock out: warehouse={warehouseId} sku={skuId}"

  - name: InventoryCacheDrift
    condition: rate(inventory_cache_drift[5m]) > 10
    severity: MEDIUM
    message: "Cache drift > 10/5min — Redis may be stale"

  - name: InventoryHoldExpiredSweep
    condition: inventory_holds_sweep_released > 50
    severity: MEDIUM
    message: "Sweep released > 50 expired holds — keyspace notifications may be broken"

  - name: OutboxRelayLag
    condition: outbox_unpublished_events > 1000
    severity: HIGH
    message: "Outbox relay lag — Kafka may be down"

  - name: HighReconciliationDiscrepancies
    condition: inventory_reconciliation_discrepancy_rate > 0.05
    severity: HIGH
    message: "Reconciliation discrepancy rate > 5% at warehouse {warehouseId}"
```

---

## Design Summary Cheat Sheet

```
QUESTION 1 — Centralized vs Distributed:
  → Centralized PostgreSQL (partitioned by warehouse_id)
  → Redis speed layer with inv:level:{w}:{sku} Hash per inventory record
  → Redis Sorted Set (inv:sku:warehouses:{sku}) for fast warehouse lookup
  → MongoDB for audit snapshots and reconciliation sessions
  → No distributed sharding: 50 warehouses × 100K SKUs fits one PG cluster

QUESTION 2 — Concurrent Inventory Updates:
  → Redis Lua atomic check-and-decrement (hold placement, zero race conditions)
  → PostgreSQL optimistic locking (version column, WHERE version = :v)
  → DB CHECK constraints as last line of defence (no negative stock ever)
  → Idempotency keys on movements (duplicate Kafka → duplicate insert rejected)
  → Retry loop (max 3 × exponential backoff) on optimistic lock failure

QUESTION 3 — Warehouse Selection Algorithm:
  → Score = 0.40·ZoneScore + 0.35·StockScore + 0.15·PriorityScore + 0.10·LoadScore
  → Redis Sorted Set (scored by available qty) narrows candidates in O(log N)
  → Zone pre-computed and cached in Redis (24h TTL)
  → Multi-SKU orders: try single-warehouse first; split only if unavoidable

QUESTION 4 — Holds vs Available:
  → on_hand = physical units in warehouse
  → on_hold = units reserved for active orders (not yet confirmed)
  → available = on_hand - on_hold (what can be sold)
  → in_transit = stock en route from transfer (pre-credited to destination)
  → Hold TTL: 15-minute Redis expiry key → keyspace event → auto-release
  → Safety-net DB sweep every 60s catches missed keyspace notifications

QUESTION 5 — Reconciliation Discrepancies:
  → Take system snapshot at count start → compare to physical counts
  → Auto-approve: |delta| ≤ 5 units OR ≤ 1% → immediate adjustment
  → Flag for review: |delta| > threshold → manager approval required
  → Resolution types: AUTO_ADJUSTMENT, MANAGER_ADJUSTMENT, WRITE_OFF, RECOUNT
  → All adjustments recorded as CYCLE_COUNT_ADJUSTMENT movement (immutable audit)

QUESTION 6 — High-Volume Accuracy:
  → 6-layer strategy: Lua atomic → optimistic lock → CHECK constraints →
    movement log verification → Redis/DB watchdog → idempotency keys
  → Movement log running-total verification (hourly job)
  → Redis/DB consistency watchdog: sample 1,000 pairs every 5 minutes, auto-heal
  → Immutable audit trail: every stock change = signed movement record

TECH DECISIONS:
  PostgreSQL  → ACID inventory_levels, holds, movements (source of truth)
  Redis       → Speed layer: holds, available stock, warehouse selection
  MongoDB     → Flexible docs: reconciliation sessions, discrepancies, snapshots
  Kafka       → inventory.movements, .holds, .transfers, .alerts, .reconcile
  Outbox      → Guaranteed event publish on every PostgreSQL transaction
  Debezium    → CDC from inventory table → downstream analytics pipelines
  Resilience4j→ Circuit breaker on Redis (fallback to DB), retry on opt-lock fail
```

---

*Stack: Java 17 · Spring Boot 3 · PostgreSQL 16 (partitioned) · Redis 7 Cluster · MongoDB 7 · Apache Kafka 3.x · Debezium · Resilience4j · Micrometer*
