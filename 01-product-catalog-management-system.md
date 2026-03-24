# Product Catalog Management System — Deep Dive Design

> **Scenario:** Retail chain with 1,000 stores and 500,000 SKUs.
> 10,000 reads/sec · 100 writes/sec · Real-time price updates · Multi-language support.
> Search with filters: category, price range, brand, ratings.

**Tech Stack:** Java 17 · Spring Boot 3 · PostgreSQL · MongoDB · Redis · Elasticsearch · Kafka · CDN

---

## Table of Contents

1. [Requirements & Constraints](#1-requirements--constraints)
2. [Capacity Estimation](#2-capacity-estimation)
3. [High-Level Architecture](#3-high-level-architecture)
4. [Core Design Questions Answered](#4-core-design-questions-answered)
   - 4.1 What microservices would you create?
   - 4.2 How would you structure the databases?
   - 4.3 How would you handle real-time price updates?
   - 4.4 How would you implement search functionality?
   - 4.5 How would you ensure catalog consistency across stores?
5. [Database Schema Design](#5-database-schema-design)
6. [Redis Data Structures](#6-redis-data-structures)
7. [Elasticsearch Index Design](#7-elasticsearch-index-design)
8. [Kafka Event Flow](#8-kafka-event-flow)
9. [CQRS Pattern — Read vs Write Separation](#9-cqrs-pattern--read-vs-write-separation)
10. [Implementation Code](#10-implementation-code)
    - 10.1 Product Service — Write Path
    - 10.2 Catalog Query Service — Read Path
    - 10.3 Price Update Pipeline
    - 10.4 Search Service
    - 10.5 Multi-Language Support
    - 10.6 Store Availability Service
11. [Failure Scenarios & Mitigations](#11-failure-scenarios--mitigations)
12. [Scaling Strategy](#12-scaling-strategy)
13. [Monitoring & Observability](#13-monitoring--observability)
14. [Summary Cheat Sheet](#14-summary-cheat-sheet)

---

## 1. Requirements & Constraints

### Functional Requirements

- Manage **500,000 SKUs** across **1,000 stores** with full product attributes (name, description, price, images, category, brand)
- Store-specific **availability and pricing** (price can differ by store/region)
- **Real-time price updates** — price change must propagate to all stores within seconds
- **Search with filters** — category, price range, brand, ratings, in-stock flag, multi-keyword
- **Multi-language support** — at minimum English + one other locale per region
- Product **categories with hierarchy** (Electronics > Phones > Smartphones)
- Image management — multiple images per product, multiple sizes/resolutions

### Non-Functional Requirements

- **Read throughput:** 10,000 reads/sec (product detail + search combined)
- **Write throughput:** 100 writes/sec (catalog updates, price changes)
- **Read latency:** p99 < 50ms for product detail; p99 < 200ms for search
- **Write latency:** p99 < 500ms for catalog update (acknowledged, async propagation)
- **Availability:** 99.99% read path; 99.9% write path
- **Consistency:** Price updates eventually consistent within 5 seconds across all stores
- **Durability:** Zero product data loss; all updates persisted before acknowledgement

### Out of Scope

- Order management, checkout, payment
- Inventory reservation (covered in separate design)
- User authentication and profiles

---

## 2. Capacity Estimation

```
Products:          500,000 SKUs
Stores:            1,000
Store-SKU pairs:   500,000 × 1,000 = 500,000,000 availability records
  → Not all SKUs in all stores; assume avg 20% = 100,000,000 active pairs

Read load:
  10,000 reads/sec
  Peak multiplier 3x = 30,000 reads/sec
  Avg product page payload = 5 KB
  → 150 MB/s read bandwidth (peak)

Write load:
  100 writes/sec (catalog writes)
  Price updates: up to 10,000/sec during promotions
    (1,000 stores × 10 price changes/sec = 10,000 price events)

Storage:
  Product core data:
    500,000 products × avg 10 KB (text + metadata) = 5 GB (PostgreSQL)

  Product images:
    500,000 products × 5 images × avg 500 KB = 1.25 TB → Object storage (S3)
    Thumbnails served via CDN

  MongoDB (flexible attributes):
    500,000 products × avg 50 KB (rich attributes, specs, translations) = 25 GB

  Elasticsearch index:
    500,000 products × ~5 KB indexed fields = 2.5 GB
    With replicas (×2) = 5 GB

  Redis cache:
    Hot products (top 10,000 by traffic) × 5 KB = 50 MB
    Price cache (all active store-SKU prices) = 100,000,000 × 20 bytes = 2 GB
    → Use Redis Cluster for price cache

Cache hit ratio target:
  Product details: 95% (top 10K products serve 80% of reads)
  Price data:      99% (always in Redis, source of truth for reads)
```

---

## 3. High-Level Architecture

```
                    ┌──────────────────────────────────────────────────────┐
                    │                 CLIENT LAYER                         │
                    │   Browser / Mobile App / Store Kiosk / Partner API  │
                    └────────────────────┬─────────────────────────────────┘
                                         │
                    ┌────────────────────▼─────────────────────────────────┐
                    │              CDN (Cloudflare / CloudFront)           │
                    │   Static assets (images, thumbnails)                 │
                    │   Edge-cached product pages (TTL 60s)                │
                    └────────────────────┬─────────────────────────────────┘
                                         │
                    ┌────────────────────▼─────────────────────────────────┐
                    │             API GATEWAY (Kong / Spring Cloud)        │
                    │   Rate limiting · JWT validation · Routing           │
                    └──────┬───────────────┬────────────────┬──────────────┘
                           │               │                │
           ┌───────────────▼──┐   ┌────────▼──────┐   ┌───▼──────────────┐
           │  PRODUCT SERVICE │   │ CATALOG QUERY │   │  SEARCH SERVICE  │
           │  (Write Path)    │   │ SERVICE       │   │  (Elasticsearch) │
           │  - CRUD ops      │   │ (Read Path)   │   │  - Full-text     │
           │  - Validation    │   │ - Cache-aside │   │  - Facets        │
           │  - Publishes     │   │ - Redis L1/L2 │   │  - Filters       │
           │    Kafka events  │   │ - Translated  │   │  - Suggestions   │
           └───────┬──────────┘   └───────────────┘   └──────────────────┘
                   │
           ┌───────▼──────────┐
           │  PRICE SERVICE   │
           │  - Store prices  │
           │  - Bulk updates  │
           │  - Promotions    │
           └───────┬──────────┘
                   │
    ┌──────────────▼──────────────────────────────────────────────────────┐
    │                         KAFKA                                       │
    │   product-events · price-events · catalog-sync · image-events      │
    └──────┬────────────────────────┬────────────────────────────────────┘
           │                        │
  ┌────────▼─────────┐    ┌─────────▼──────────┐   ┌──────────────────────┐
  │ SEARCH INDEXER   │    │  STORE SYNC SVC    │   │ AVAILABILITY SERVICE │
  │ (Kafka consumer) │    │  (Kafka consumer)  │   │  - Per-store stock   │
  │ Updates ES index │    │  Pushes to stores  │   │  - Open/closed       │
  └──────────────────┘    └────────────────────┘   └──────────────────────┘

Data Stores:
  ┌──────────────┐  ┌──────────────┐  ┌────────────────┐  ┌──────────────┐
  │ POSTGRESQL   │  │   MONGODB    │  │     REDIS      │  │ELASTICSEARCH │
  │ Core catalog │  │ Rich attrs   │  │ Price cache    │  │ Search index │
  │ Categories   │  │ Translations │  │ Product cache  │  │ Facets       │
  │ Price rules  │  │ Specs/dims   │  │ Store avail.   │  │ Suggestions  │
  │ Store config │  │ Image meta   │  │ Rate limits    │  │              │
  └──────────────┘  └──────────────┘  └────────────────┘  └──────────────┘
                                      ┌──────────────────────────────────┐
                                      │     S3 / OBJECT STORAGE          │
                                      │     Product images (original)    │
                                      │     CDN origin for thumbnails    │
                                      └──────────────────────────────────┘
```

---

## 4. Core Design Questions Answered

---

### 4.1 What Microservices Would You Create?

The decomposition follows **business capability boundaries** — each service owns one domain concept end-to-end.

```
Service                  Responsibility                         Owns
────────────────────────────────────────────────────────────────────────────
product-service          Core CRUD for products (write path)   PostgreSQL +
                         Validation, enrichment, events         MongoDB

catalog-query-service    Read-optimized product retrieval       Redis cache
                         Multi-language response assembly        (no DB writes)

price-service            Store-level pricing, promotions        PostgreSQL
                         Bulk price updates, price rules        Redis (hot prices)

search-service           Full-text search, filters, facets      Elasticsearch
                         Suggestions, synonyms, rankings

availability-service     Per-store stock levels (read)          Redis + MongoDB
                         In-stock/out-of-stock per location

category-service         Category tree management               PostgreSQL
                         Breadcrumbs, hierarchy navigation

image-service            Image upload, resizing, CDN            S3 + CDN
                         Thumbnail generation, metadata

store-sync-service       Propagate catalog changes to stores    Kafka consumer
                         Handles store-specific overrides

translation-service      Multi-language product content         MongoDB
                         Locale management, fallback chains
```

**Communication rules:**
- **Synchronous (REST/gRPC):** Only for real-time reads where caller needs the response immediately (e.g., catalog-query-service fetching a product)
- **Asynchronous (Kafka):** All write-side state changes (product created/updated, price changed) — no service directly calls another's write API

---

### 4.2 How Would You Structure the Databases?

**Three databases serve different access patterns — not one database trying to do everything.**

```
PostgreSQL — The System of Record (Source of Truth)
  Why: ACID transactions, relational integrity, complex queries across tables
  Stores: product core, categories, price rules, store configurations

MongoDB — The Flexible Attribute Store
  Why: Product specs vary wildly by category (a phone has RAM/storage,
       a shirt has color/size). Rigid SQL schema can't handle this well.
  Stores: rich product attributes, translations, image metadata, custom specs

Redis — The Speed Layer
  Why: 10,000 reads/sec → even fast PostgreSQL can't serve this alone.
       Redis serves cached product data in <1ms vs 10ms from DB.
  Stores: hot product cache, all store prices (key-value), availability flags

Elasticsearch — The Search Engine
  Why: PostgreSQL LIKE queries on 500K products are too slow.
       Elasticsearch provides sub-200ms full-text + faceted search.
  Stores: denormalized product documents optimized for search queries

S3 + CDN — The Binary Store
  Why: Databases are not designed for large binary files.
  Stores: product images served via CDN edge caches globally
```

**Which database wins for which query:**

| Query | Winner | Reason |
|---|---|---|
| Get product by ID | Redis → PostgreSQL fallback | Speed; Redis hit ~95% |
| Search "red nike shoes under $100" | Elasticsearch | Full-text + facet |
| Get product translations | MongoDB | Schema-flexible locale docs |
| Get price for SKU at store X | Redis | O(1) key lookup, always fresh |
| Category hierarchy navigation | PostgreSQL | Recursive CTE on tree |
| Bulk price update (100K SKUs) | PostgreSQL + Kafka → Redis | Write durability first |
| Store availability count | Redis | Sub-ms hash field lookup |

---

### 4.3 How Would You Handle Real-Time Price Updates?

**The challenge:** 100 writes/sec for catalog but price promotions can trigger **10,000 price events/sec** (1,000 stores × 10 updates). Must propagate to all reads within ~5 seconds.

**Design: Write to PostgreSQL for durability → Kafka for fan-out → Redis as the price read cache.**

```
Price Update Flow:

1. Operator sets new price via Admin UI
   POST /api/v1/prices
   Body: { saleId, storeId (or "*" for all), priceCents, effectiveAt }

2. Price Service writes to PostgreSQL (durable, transactional)
   INSERT INTO price_rules (sku_id, store_id, price_cents, effective_at)
   → Returns 200 OK immediately

3. Price Service publishes Kafka event
   Topic: price-events
   Key: skuId (partitioned for ordering per SKU)
   Value: { skuId, storeId, oldPrice, newPrice, effectiveAt, eventId }

4. Redis Price Updater (Kafka consumer) updates Redis atomically
   For storeId = "*" (all stores):
     Pipeline 1,000 HSET commands into one Redis pipeline
     Key: "price:{skuId}" → Hash field per store → priceCents value
     1,000 HSET ops in ~10ms via pipeline

5. Cache Invalidator (Kafka consumer) deletes cached product responses
   DEL "product:{skuId}:*"  (wildcard delete or tag-based invalidation)
   Forces next reads to re-assemble with fresh price

6. CDN Purge (Kafka consumer) — optional
   Trigger CDN cache purge for product page URLs
   Only needed if product pages are fully edge-cached

Total propagation time:
  PostgreSQL write:      5ms
  Kafka publish:         5ms
  Kafka consumer lag:    <100ms (p99)
  Redis pipeline write:  10ms
  ───────────────────────────
  End-to-end:            ~120ms  ✅ (well within 5-second SLA)
```

**Bulk price update (e.g., 20% Black Friday discount on all 500K products):**

```
Problem: 500,000 × 1,000 stores = 500M Redis writes would take too long serially

Solution: Tiered fan-out with Kafka partitioning
  1. Admin creates a "promotion rule" in PostgreSQL
     { promotion_id, discount_pct: 20, applies_to: ALL, starts_at: T }

  2. Price Service publishes 500K Kafka messages (one per SKU), batched
     Kafka handles fan-out; 50 partitions × 10K msgs each

  3. Redis Updater consumers (scale to 50 instances) each consume 10K messages
     Each writes ~10K × 1,000 = 10M HSET to Redis → done in ~30 seconds total

  4. Redis TTL trick: Instead of writing all prices,
     store promotion rules in Redis separately
     Price read path = base price × active promotions
     Reduces writes from 500M to 500K + 1 rule record
```

**Price read path (how a product page gets the current price):**

```java
// Fast price lookup — Redis HGET, never hits DB on reads
public long getPrice(String skuId, String storeId) {
    // Try store-specific price first
    String storePrice = redis.opsForHash().get("price:" + skuId, storeId);
    if (storePrice != null) return Long.parseLong(storePrice);

    // Fallback to regional/national price
    String nationalPrice = redis.opsForHash().get("price:" + skuId, "NATIONAL");
    if (nationalPrice != null) return Long.parseLong(nationalPrice);

    // Last resort: DB (very rare — only if Redis is cold)
    return priceRepository.findCurrentPrice(skuId, storeId);
}
```

---

### 4.4 How Would You Implement Search Functionality?

**Technology: Elasticsearch** — purpose-built for full-text search with sub-200ms responses on 500K products.

**PostgreSQL LIKE '%keyword%' is not an option at scale:**
```
PostgreSQL LIKE '%red shoes%' on 500K products:
  → Full table scan: 2–5 seconds
  → No ranking by relevance
  → No faceted counts (how many products per brand)
  → No phonetic matching ("Nkie" won't find "Nike")

Elasticsearch on same query:
  → Inverted index lookup: 20–80ms
  → BM25 relevance ranking built in
  → Facet aggregations in same query
  → Fuzzy matching, synonyms, stop words
```

**Search index design (see Section 7 for full mapping):**

```
One Elasticsearch document per product:
  - Searchable text fields: name, description, brand, tags (analyzed)
  - Filter fields: categoryId, brandId, storeIds (keyword, not analyzed)
  - Range fields: priceCents, averageRating, reviewCount (numeric)
  - Geo field: storeLocations[] (geo_point for "near me" search)
  - Boost fields: salesRank, isPremiumListing (affect ranking)

Search query decomposition:
  User types: "red nike running shoes under $100 size 10"

  NLP layer extracts:
    keyword:   "nike running shoes"
    color:     "red"       → filter on attributes.color
    brand:     "nike"      → filter on brand keyword field
    maxPrice:  10000       → range filter on priceCents
    attributes: size=10    → filter on attributes.size

  Elasticsearch query:
    must: multi_match("nike running shoes", fields: [name^3, brand^2, description])
    filter: term(brand, "nike")
    filter: term(attributes.color, "red")
    filter: range(priceCents, lte: 10000)
    filter: term(attributes.size, "10")
    aggs: terms(categoryId), terms(brandId), range(priceCents, [0-50, 50-100, 100+])
```

**Search-as-you-type (suggestions):**

```
Elasticsearch completion suggester:
  Field: "suggest" with type "completion"
  Input: ["Nike Running Shoes", "Nike Air Max", "Nike Air Force"]
  Queried with prefix match → returns in <5ms

  GET /catalog/_search/suggest
  {
    "product_suggest": {
      "prefix": "nike ru",
      "completion": { "field": "suggest", "size": 5, "fuzzy": { "fuzziness": 1 } }
    }
  }
```

**Keeping Elasticsearch in sync:**

```
Write path: product-service → Kafka → search-indexer → Elasticsearch
  product-service does NOT call Elasticsearch directly (decoupled)
  search-indexer is a Kafka consumer that:
    1. Reads ProductUpdatedEvent
    2. Fetches full product from product-service (or MongoDB)
    3. Assembles search document (denormalized: includes category name, brand name)
    4. Calls Elasticsearch bulk index API
    5. At-most-once lag: 1–3 seconds after write

Bulk re-index (e.g., after ES mapping change):
  Use index aliases: "catalog" alias → "catalog_v2" (new index)
  Re-index in background: product-service exports to Kafka topic "catalog-full-sync"
  search-indexer consumes and builds new index
  When complete: alias swap (zero-downtime)
  curl -XPOST /aliases: { "add": {"index":"catalog_v3", "alias":"catalog"} }
```

---

### 4.5 How Would You Ensure Catalog Consistency Across 1,000 Stores?

**The core tension:** Strong consistency requires every store to confirm receipt before acknowledging the write — too slow. Pure eventual consistency risks stores showing different prices indefinitely.

**Solution: Write-through durability + Kafka fan-out + version-stamped events + monotonic read guarantee in Redis.**

```
Consistency Model: "Bounded Eventual Consistency" — all stores converge within 5 seconds.

Key mechanisms:

1. Event Versioning
   Every product/price update carries a version number (monotonic)
   product-service increments version on each update (PostgreSQL sequence)
   Kafka message includes version: { skuId, version: 42, ...payload }

   Store-sync-service tracks last-seen version per store per SKU:
   Redis key: "store_version:{storeId}:{skuId}" = 42
   If incoming event version ≤ stored version → duplicate, skip (idempotent)

2. Ordered delivery per SKU
   Kafka partition key = skuId
   All events for same SKU → same partition → total ordering guaranteed
   No risk of applying version 43 before version 42 for the same product

3. Store acknowledgement tracking (optional for SLA compliance)
   For high-value price changes (>20% discount), require stores to confirm:
   Kafka consumer in each store sends ACK to "store-ack" topic
   store-sync-service tracks per-store confirmation
   Dashboard: "Store consistency: 998/1000 stores updated"

4. Reconciliation job (catch-up for stores that were offline)
   Every 5 minutes: compare store's last-seen version vs current version
   If delta > 0: push missing events in order
   Stores that come back online catch up automatically

5. Read-your-writes for catalog admins
   Admin writes product update → gets back version number
   Admin reads product → query includes "min_version=42" header
   catalog-query-service checks Redis version stamp
   If Redis version < 42 → fetch directly from PostgreSQL (strong read)
   Guarantees admin sees their own update immediately
```

**Store-level override system:**

```
National catalog:     SKU-123, name="iPhone 15 Pro", price=$999
Store-level override: SKU-123, store=NYC-001, price=$979 (local promotion)
Store-level override: SKU-123, store=NYC-001, available=false (out of stock)

Resolution order (highest priority wins):
  1. Store-specific override (store+SKU match)
  2. Regional override (region+SKU match)
  3. National default (SKU only)

Stored in Redis as a Hash per SKU:
  HSET product:123:overrides  NYC-001  '{"price":97900,"available":false}'
  HSET product:123:overrides  EAST-REGION '{"price":98900}'
  HSET product:123:overrides  NATIONAL '{"price":99900,"available":true}'

Read path assembles the final product view per store in <2ms:
  base = HGET product:123:overrides NATIONAL
  regional = HGET product:123:overrides {storeRegion}
  local = HGET product:123:overrides {storeId}
  → merge(base, regional, local) → final product for this store
```

---

## 5. Database Schema Design

### PostgreSQL — Core Catalog (Source of Record)

```sql
-- Category hierarchy (adjacency list + materialized path for fast traversal)
CREATE TABLE categories (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    parent_id   UUID REFERENCES categories(id),
    name        VARCHAR(200) NOT NULL,
    slug        VARCHAR(200) UNIQUE NOT NULL,
    path        TEXT NOT NULL,    -- e.g. '/electronics/phones/smartphones'
    depth       INT NOT NULL DEFAULT 0,
    sort_order  INT NOT NULL DEFAULT 0,
    is_active   BOOLEAN NOT NULL DEFAULT true,
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_categories_parent ON categories(parent_id);
CREATE INDEX idx_categories_path   ON categories USING GIST(path gist_trgm_ops);

-- Core product table (lean — rich attributes in MongoDB)
CREATE TABLE products (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sku             VARCHAR(100) UNIQUE NOT NULL,
    category_id     UUID NOT NULL REFERENCES categories(id),
    brand_id        UUID NOT NULL REFERENCES brands(id),
    name            VARCHAR(500) NOT NULL,           -- default locale
    slug            VARCHAR(500) UNIQUE NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'DRAFT',
                    -- DRAFT | ACTIVE | DISCONTINUED | ARCHIVED
    base_price_cents BIGINT NOT NULL,
    version         BIGINT NOT NULL DEFAULT 1,       -- monotonic update counter
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_products_sku        ON products(sku);
CREATE INDEX idx_products_category   ON products(category_id);
CREATE INDEX idx_products_brand      ON products(brand_id);
CREATE INDEX idx_products_status     ON products(status);
CREATE INDEX idx_products_updated_at ON products(updated_at DESC); -- sync queries

-- Brand table
CREATE TABLE brands (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name        VARCHAR(200) UNIQUE NOT NULL,
    slug        VARCHAR(200) UNIQUE NOT NULL,
    logo_url    TEXT,
    is_active   BOOLEAN NOT NULL DEFAULT true
);

-- Price rules (store-specific pricing)
CREATE TABLE price_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id      UUID NOT NULL REFERENCES products(id),
    store_id        UUID,           -- NULL = national/default price
    region_id       UUID,           -- NULL = not region-specific
    price_cents     BIGINT NOT NULL,
    promotion_id    UUID,           -- NULL = not part of promotion
    effective_from  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    effective_to    TIMESTAMPTZ,    -- NULL = no expiry
    created_at      TIMESTAMPTZ DEFAULT NOW(),

    -- Only one active rule per product+store at a time
    CONSTRAINT unique_active_rule UNIQUE (product_id, store_id, effective_from)
);

CREATE INDEX idx_price_rules_product  ON price_rules(product_id);
CREATE INDEX idx_price_rules_store    ON price_rules(store_id);
CREATE INDEX idx_price_rules_eff_from ON price_rules(effective_from);

-- Store configuration
CREATE TABLE stores (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code        VARCHAR(20) UNIQUE NOT NULL,
    name        VARCHAR(200) NOT NULL,
    region_id   UUID REFERENCES regions(id),
    timezone    VARCHAR(50) NOT NULL,
    currency    CHAR(3) NOT NULL DEFAULT 'USD',
    locale      VARCHAR(10) NOT NULL DEFAULT 'en-US',
    is_active   BOOLEAN NOT NULL DEFAULT true
);

-- Image metadata (binaries in S3)
CREATE TABLE product_images (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id  UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    s3_key      TEXT NOT NULL,
    cdn_url     TEXT NOT NULL,
    alt_text    VARCHAR(500),
    width_px    INT,
    height_px   INT,
    file_size   BIGINT,
    sort_order  INT NOT NULL DEFAULT 0,
    is_primary  BOOLEAN NOT NULL DEFAULT false,
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_images_product ON product_images(product_id, sort_order);
```

### MongoDB — Rich Product Attributes & Translations

```javascript
// Product attributes document (one per SKU)
// Collection: product_attributes
{
  _id: ObjectId,
  productId: "uuid-of-product",   // references PostgreSQL products.id
  sku: "APPLE-IPHONE-15-PRO-256",
  version: 42,                     // matches PostgreSQL version column

  // Flexible specs — schema varies wildly by category
  specifications: {
    // Electronics
    display: { size: "6.1 inch", resolution: "2556×1179", type: "Super Retina XDR" },
    battery: { capacity: "3274 mAh", chargingWatts: 20 },
    storage: [{ capacity: "128GB", sku: "IPHONE-128" }, { capacity: "256GB", sku: "IPHONE-256" }],
    connectivity: ["5G", "WiFi 6E", "Bluetooth 5.3", "NFC"],

    // Apparel (different category, completely different specs)
    // sizes: ["XS","S","M","L","XL"], materials: ["100% Cotton"]
  },

  // Multi-language translations
  translations: {
    "en-US": {
      name: "iPhone 15 Pro",
      description: "The most powerful iPhone ever...",
      shortDescription: "Titanium design, A17 Pro chip",
      searchKeywords: ["iphone", "apple", "smartphone", "ios"],
    },
    "es-MX": {
      name: "iPhone 15 Pro",
      description: "El iPhone más poderoso jamás creado...",
      shortDescription: "Diseño de titanio, chip A17 Pro",
      searchKeywords: ["iphone", "apple", "teléfono inteligente"],
    },
    "fr-FR": {
      name: "iPhone 15 Pro",
      description: "L'iPhone le plus puissant jamais créé...",
      shortDescription: "Design en titane, puce A17 Pro",
      searchKeywords: ["iphone", "apple", "smartphone"],
    }
  },

  // SEO metadata
  seo: {
    "en-US": { title: "iPhone 15 Pro - Buy Online | RetailChain", metaDescription: "..." },
    "es-MX": { title: "iPhone 15 Pro - Compra en Línea | RetailChain", metaDescription: "..." }
  },

  // Variant information
  variants: [
    { sku: "IPHONE-15-PRO-128-BLACK", color: "Black Titanium", storage: "128GB" },
    { sku: "IPHONE-15-PRO-256-BLUE",  color: "Blue Titanium",  storage: "256GB" }
  ],

  updatedAt: ISODate("2026-01-15T10:30:00Z")
}

// Indexes for MongoDB
db.product_attributes.createIndex({ "productId": 1 }, { unique: true })
db.product_attributes.createIndex({ "sku": 1 }, { unique: true })
db.product_attributes.createIndex({ "version": 1 })
db.product_attributes.createIndex({ "updatedAt": -1 })
// Text search index (fallback if ES is unavailable)
db.product_attributes.createIndex({
  "translations.en-US.name": "text",
  "translations.en-US.description": "text",
  "sku": "text"
})
```

---

## 6. Redis Data Structures

```
Key Pattern                                  Type     TTL        Purpose
─────────────────────────────────────────────────────────────────────────────
product:{id}                                 STRING   30 min     Full product JSON (assembled response)
product:{id}:v{version}                      STRING   30 min     Versioned cache (read-your-writes)
product:slug:{slug}                          STRING   30 min     Slug → ID mapping
product:{id}:images                          LIST     1 hour     Image URL list (sorted)

price:{skuId}                                HASH     None       All store prices
  field: {storeId} or "NATIONAL"
  value: price in cents

price:{skuId}:promotions                     SET      Until exp  Active promotions for this SKU

category:tree                                STRING   24 hours   Full serialized category tree JSON
category:{id}:children                       SET      1 hour     Direct child category IDs
category:{id}:breadcrumb                     STRING   1 hour     Breadcrumb JSON

availability:{skuId}:{storeId}               STRING   5 min      "IN_STOCK" | "OUT_OF_STOCK" | "LIMITED"
availability:{skuId}:stores                  SET      5 min      Set of storeIds where SKU is available

search:suggest:{prefix}                      STRING   5 min      Cached suggestion results
search:popular                               ZSET     1 hour     Top searches (score = search count)

store:{storeId}:meta                         STRING   1 hour     Store config JSON (timezone, locale)
store:{storeId}:version:{skuId}              STRING   24 hours   Last-seen event version per store/SKU

rate_limit:write:{userId}                    STRING   1 min      Write rate limiting per admin user
```

---

## 7. Elasticsearch Index Design

```json
{
  "mappings": {
    "properties": {
      "productId":    { "type": "keyword" },
      "sku":          { "type": "keyword" },

      "name": {
        "type": "text",
        "analyzer": "english",
        "fields": {
          "keyword": { "type": "keyword" },
          "suggest": { "type": "completion" }
        }
      },

      "description":  { "type": "text", "analyzer": "english" },
      "brand": {
        "type": "text", "analyzer": "english",
        "fields": { "keyword": { "type": "keyword" } }
      },

      "categoryId":   { "type": "keyword" },
      "categoryPath": { "type": "keyword" },
      "categoryName": { "type": "text", "analyzer": "english" },

      "priceCents":       { "type": "long" },
      "averageRating":    { "type": "float" },
      "reviewCount":      { "type": "integer" },
      "salesRank":        { "type": "integer" },
      "status":           { "type": "keyword" },
      "isPremium":        { "type": "boolean" },

      "availableStoreIds": { "type": "keyword" },

      "attributes": {
        "type": "nested",
        "properties": {
          "key":   { "type": "keyword" },
          "value": { "type": "keyword" }
        }
      },

      "tags":     { "type": "keyword" },
      "imageUrl": { "type": "keyword", "index": false },

      "translations": {
        "type": "object",
        "dynamic": true
      },

      "updatedAt": { "type": "date" }
    }
  },

  "settings": {
    "number_of_shards":   5,
    "number_of_replicas": 1,
    "analysis": {
      "analyzer": {
        "english": {
          "tokenizer": "standard",
          "filter": ["lowercase", "english_stop", "english_stemmer", "synonym_filter"]
        }
      },
      "filter": {
        "synonym_filter": {
          "type": "synonym",
          "synonyms": ["mobile,phone,smartphone", "tv,television", "laptop,notebook"]
        },
        "english_stemmer": { "type": "stemmer", "language": "english" },
        "english_stop":    { "type": "stop",    "stopwords": "_english_" }
      }
    }
  }
}
```

**Example search query (faceted search with filters):**

```json
{
  "query": {
    "bool": {
      "must": [{
        "multi_match": {
          "query": "running shoes",
          "fields": ["name^3", "brand^2", "description", "tags^1.5"],
          "fuzziness": "AUTO"
        }
      }],
      "filter": [
        { "term":  { "status": "ACTIVE" } },
        { "term":  { "brand.keyword": "Nike" } },
        { "range": { "priceCents": { "gte": 5000, "lte": 15000 } } },
        { "nested": {
            "path": "attributes",
            "query": {
              "bool": { "must": [
                { "term": { "attributes.key": "size" } },
                { "term": { "attributes.value": "10" } }
              ]}
            }
          }
        },
        { "term": { "availableStoreIds": "NYC-001" } }
      ]
    }
  },
  "aggs": {
    "by_category": { "terms": { "field": "categoryId", "size": 10 } },
    "by_brand":    { "terms": { "field": "brand.keyword", "size": 20 } },
    "price_ranges": {
      "range": {
        "field": "priceCents",
        "ranges": [
          { "to": 5000 },
          { "from": 5000, "to": 10000 },
          { "from": 10000, "to": 25000 },
          { "from": 25000 }
        ]
      }
    },
    "avg_rating": { "avg": { "field": "averageRating" } }
  },
  "sort": [
    { "_score": "desc" },
    { "salesRank": "asc" }
  ],
  "from": 0, "size": 24
}
```

---

## 8. Kafka Event Flow

```
Topic                   Partitions   Key           Retention   Consumers
──────────────────────────────────────────────────────────────────────────────
product-events          20           productId     7 days      search-indexer
                                                               store-sync-service
                                                               cache-invalidator
                                                               analytics-service

price-events            50           skuId         7 days      redis-price-updater
                                                               store-sync-service
                                                               search-indexer (price field)
                                                               analytics-service

catalog-sync            10           storeId       3 days      store-facing consumers
                                                               (store POS systems)

image-events            5            productId     3 days      image-resizer
                                                               cdn-invalidator

category-events         3            categoryId    30 days     all read caches
```

**Event schemas:**

```java
// ProductUpdatedEvent — published on any product write
record ProductUpdatedEvent(
    String eventId,           // UUID for idempotency
    String productId,
    String sku,
    long   version,           // monotonic version from PostgreSQL
    String eventType,         // CREATED | UPDATED | DISCONTINUED | ARCHIVED
    String updatedBy,
    Instant occurredAt,
    Map<String, Object> changedFields   // only the diff, not full product
) {}

// PriceChangedEvent — published on any price write
record PriceChangedEvent(
    String eventId,
    String skuId,
    String storeId,           // null = national price
    String regionId,          // null = not regional
    long   oldPriceCents,
    long   newPriceCents,
    String promotionId,       // null = regular price
    Instant effectiveAt,
    Instant occurredAt
) {}
```

---

## 9. CQRS Pattern — Read vs Write Separation

The 100:1 read-to-write ratio makes CQRS (Command Query Responsibility Segregation) a natural fit. Write models optimize for correctness; read models optimize for speed.

```
WRITE PATH (Command Side):
  POST/PUT/PATCH → product-service → PostgreSQL (normalized, ACID)
                                   → MongoDB (flexible attributes)
                                   → Kafka (state change events)

  product-service does NOT build the read response —
  it just persists and publishes events.

READ PATH (Query Side):
  GET → catalog-query-service → Redis L1 cache (hit 95%)
                              → MongoDB (miss 5%, for rich attributes)
                              → PostgreSQL (rare, strong-read requests)

  catalog-query-service assembles the response:
    1. Core fields from Redis (cached from PostgreSQL)
    2. Rich attributes from MongoDB
    3. Store price from Redis price hash
    4. Availability from Redis availability key
    5. Translations for requested locale from MongoDB

  This "assembly" pattern means:
  → No JOIN queries at read time
  → Each data source serves only what it's best at
  → Read service is stateless and horizontally scalable
```

**Read model assembly (catalog-query-service):**

```
Request: GET /api/v1/products/APPLE-IPHONE-15-PRO?storeId=NYC-001&locale=en-US

Assembly steps (parallel where possible):
  ┌─ Step 1: Redis GET product:{id}          → core fields (1ms)
  ├─ Step 2: Redis HGET price:{sku} NYC-001  → store price (1ms)   ┐ parallel
  ├─ Step 3: Redis GET availability:{sku}:{store} → stock (1ms)    │
  └─ Step 4: Redis LRANGE product:{id}:images → image URLs (1ms)   ┘

  If Step 1 miss → MongoDB findOne({productId}) → populate cache (5ms)

  Step 5: Merge translations for locale=en-US from MongoDB doc
          (if full product was fetched; otherwise from separate cache)

Total read time (cache hit): ~2ms (parallel Redis calls)
Total read time (cache miss): ~8ms (MongoDB fetch + Redis populate)
```

---

## 10. Implementation Code

### 10.1 Product Service — Write Path

```java
@RestController
@RequestMapping("/api/v1/products")
@RequiredArgsConstructor
public class ProductController {

    private final ProductCommandService productCommandService;

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public ProductCreatedResponse createProduct(
            @RequestBody @Valid CreateProductRequest request,
            @RequestHeader("X-Admin-UserId") String adminUserId) {
        return productCommandService.createProduct(request, adminUserId);
    }

    @PutMapping("/{productId}")
    public ProductResponse updateProduct(
            @PathVariable String productId,
            @RequestBody @Valid UpdateProductRequest request,
            @RequestHeader("X-Admin-UserId") String adminUserId) {
        return productCommandService.updateProduct(productId, request, adminUserId);
    }

    @PatchMapping("/{productId}/status")
    public void updateStatus(
            @PathVariable String productId,
            @RequestParam ProductStatus status,
            @RequestHeader("X-Admin-UserId") String adminUserId) {
        productCommandService.updateStatus(productId, status, adminUserId);
    }
}

@Service
@RequiredArgsConstructor
@Transactional
public class ProductCommandService {

    private final ProductRepository productRepo;          // PostgreSQL JPA
    private final ProductAttributeRepository attrRepo;   // MongoDB
    private final KafkaTemplate<String, Object> kafka;
    private final StringRedisTemplate redis;

    public ProductCreatedResponse createProduct(CreateProductRequest req, String adminUserId) {
        // 1. Save core product to PostgreSQL
        Product product = Product.builder()
            .sku(req.getSku())
            .categoryId(req.getCategoryId())
            .brandId(req.getBrandId())
            .name(req.getName())
            .slug(slugify(req.getName()))
            .status(ProductStatus.DRAFT)
            .basePriceCents(req.getBasePriceCents())
            .version(1L)
            .build();

        product = productRepo.save(product);  // auto-generates UUID

        // 2. Save rich attributes to MongoDB
        ProductAttributes attributes = ProductAttributes.builder()
            .productId(product.getId())
            .sku(product.getSku())
            .version(product.getVersion())
            .specifications(req.getSpecifications())
            .translations(req.getTranslations())
            .variants(req.getVariants())
            .build();

        attrRepo.save(attributes);

        // 3. Set national price in Redis
        redis.opsForHash().put("price:" + product.getSku(), "NATIONAL",
            String.valueOf(product.getBasePriceCents()));

        // 4. Publish event to Kafka (triggers ES indexing, store sync, etc.)
        kafka.send("product-events", product.getId(),
            new ProductUpdatedEvent(
                UUID.randomUUID().toString(),
                product.getId(),
                product.getSku(),
                product.getVersion(),
                "CREATED",
                adminUserId,
                Instant.now(),
                Map.of("status", "DRAFT", "sku", product.getSku())
            ));

        return new ProductCreatedResponse(product.getId(), product.getSku(), product.getVersion());
    }

    public ProductResponse updateProduct(String productId, UpdateProductRequest req, String adminUserId) {
        Product product = productRepo.findById(productId)
            .orElseThrow(() -> new ProductNotFoundException(productId));

        // Track only changed fields for the event
        Map<String, Object> changedFields = new LinkedHashMap<>();

        if (req.getName() != null && !req.getName().equals(product.getName())) {
            product.setName(req.getName());
            product.setSlug(slugify(req.getName()));
            changedFields.put("name", req.getName());
        }
        if (req.getBasePriceCents() != null && !req.getBasePriceCents().equals(product.getBasePriceCents())) {
            product.setBasePriceCents(req.getBasePriceCents());
            changedFields.put("basePriceCents", req.getBasePriceCents());
        }

        if (changedFields.isEmpty()) {
            return toResponse(product); // no-op
        }

        // Increment version atomically
        int updated = productRepo.incrementVersion(productId, product.getVersion());
        if (updated == 0) throw new ConcurrentModificationException("Product modified concurrently");

        product.setVersion(product.getVersion() + 1);
        productRepo.save(product);

        // Update MongoDB attributes if provided
        if (req.getSpecifications() != null || req.getTranslations() != null) {
            attrRepo.updateByProductId(productId, req.getSpecifications(), req.getTranslations(), product.getVersion());
        }

        // Invalidate Redis cache
        redis.delete("product:" + productId);
        redis.delete("product:slug:" + product.getSlug());

        // Publish event
        kafka.send("product-events", productId,
            new ProductUpdatedEvent(UUID.randomUUID().toString(), productId, product.getSku(),
                product.getVersion(), "UPDATED", adminUserId, Instant.now(), changedFields));

        return toResponse(product);
    }
}
```

### 10.2 Catalog Query Service — Read Path (Cache-Aside with Two Levels)

```java
@RestController
@RequestMapping("/api/v1/catalog")
@RequiredArgsConstructor
public class CatalogQueryController {

    private final CatalogQueryService queryService;

    @GetMapping("/products/{productId}")
    public ResponseEntity<ProductDetailResponse> getProduct(
            @PathVariable String productId,
            @RequestParam(defaultValue = "en-US") String locale,
            @RequestParam(required = false) String storeId) {

        ProductDetailResponse response = queryService.getProductDetail(productId, locale, storeId);
        return ResponseEntity.ok()
            .header("Cache-Control", "public, max-age=60")  // CDN caching
            .header("X-Version", String.valueOf(response.getVersion()))
            .body(response);
    }

    @GetMapping("/products/by-slug/{slug}")
    public ResponseEntity<ProductDetailResponse> getProductBySlug(
            @PathVariable String slug,
            @RequestParam(defaultValue = "en-US") String locale,
            @RequestParam(required = false) String storeId) {

        String productId = queryService.resolveSlug(slug);
        return getProduct(productId, locale, storeId);
    }
}

@Service
@RequiredArgsConstructor
public class CatalogQueryService {

    private final StringRedisTemplate redis;
    private final ProductAttributeRepository mongoRepo;
    private final ProductRepository pgRepo;         // only for cache miss fallback
    private final ObjectMapper objectMapper;

    // L1 local cache (Caffeine) — sub-millisecond for truly hot products
    private final Cache<String, ProductDetailResponse> localCache =
        Caffeine.newBuilder()
            .maximumSize(5_000)
            .expireAfterWrite(Duration.ofSeconds(30))
            .recordStats()
            .build();

    public ProductDetailResponse getProductDetail(String productId, String locale, String storeId) {
        String cacheKey = "product:" + productId + ":" + locale + (storeId != null ? ":" + storeId : "");

        // L1: Caffeine (in-process)
        ProductDetailResponse cached = localCache.getIfPresent(cacheKey);
        if (cached != null) return cached;

        // L2: Redis
        String json = redis.opsForValue().get(cacheKey);
        if (json != null) {
            ProductDetailResponse response = deserialize(json);
            localCache.put(cacheKey, response);
            return response;
        }

        // L3: Assemble from sources (cache miss)
        ProductDetailResponse response = assembleProduct(productId, locale, storeId);

        // Populate both caches
        redis.opsForValue().set(cacheKey, serialize(response), Duration.ofMinutes(30));
        localCache.put(cacheKey, response);

        return response;
    }

    private ProductDetailResponse assembleProduct(String productId, String locale, String storeId) {
        // Parallel fetch from multiple sources
        CompletableFuture<ProductAttributes> attrFuture =
            CompletableFuture.supplyAsync(() -> mongoRepo.findByProductId(productId));

        CompletableFuture<Long> priceFuture = storeId != null
            ? CompletableFuture.supplyAsync(() -> getPrice(productId, storeId))
            : CompletableFuture.completedFuture(null);

        CompletableFuture<String> availFuture = storeId != null
            ? CompletableFuture.supplyAsync(() -> getAvailability(productId, storeId))
            : CompletableFuture.completedFuture("UNKNOWN");

        // Wait for all
        ProductAttributes attrs = attrFuture.join();
        Long price = priceFuture.join();
        String availability = availFuture.join();

        // Get core fields from PostgreSQL (only needed on true cold miss)
        Product core = pgRepo.findById(productId).orElseThrow();

        // Select translation for requested locale (with fallback chain)
        Translation translation = selectTranslation(attrs.getTranslations(), locale);

        return ProductDetailResponse.builder()
            .productId(productId)
            .sku(core.getSku())
            .version(core.getVersion())
            .name(translation.getName())
            .description(translation.getDescription())
            .brand(core.getBrand().getName())
            .categoryId(core.getCategoryId())
            .priceCents(price != null ? price : core.getBasePriceCents())
            .availability(availability)
            .specifications(attrs.getSpecifications())
            .images(getImages(productId))
            .variants(attrs.getVariants())
            .locale(locale)
            .build();
    }

    private long getPrice(String skuId, String storeId) {
        // Try store-specific → regional → national fallback
        Object storePrice = redis.opsForHash().get("price:" + skuId, storeId);
        if (storePrice != null) return Long.parseLong(storePrice.toString());

        Object nationalPrice = redis.opsForHash().get("price:" + skuId, "NATIONAL");
        if (nationalPrice != null) return Long.parseLong(nationalPrice.toString());

        // Absolute fallback — DB
        return pgRepo.findBasePriceCents(skuId);
    }

    private Translation selectTranslation(Map<String, Translation> translations, String locale) {
        // Exact match → language match (en-US → en) → fallback to en-US
        if (translations.containsKey(locale)) return translations.get(locale);
        String lang = locale.split("-")[0];
        return translations.entrySet().stream()
            .filter(e -> e.getKey().startsWith(lang))
            .findFirst()
            .map(Map.Entry::getValue)
            .orElse(translations.getOrDefault("en-US", Translation.empty()));
    }
}
```

### 10.3 Price Update Pipeline

```java
@RestController
@RequestMapping("/api/v1/prices")
@RequiredArgsConstructor
public class PriceController {

    private final PriceCommandService priceService;

    // Single store price update
    @PutMapping
    public void updatePrice(@RequestBody @Valid PriceUpdateRequest request) {
        priceService.updatePrice(request);
    }

    // Bulk price update (e.g., promotional discount)
    @PostMapping("/bulk")
    @ResponseStatus(HttpStatus.ACCEPTED)   // async — returns immediately
    public BulkUpdateJobResponse bulkUpdatePrices(@RequestBody @Valid BulkPriceUpdateRequest request) {
        return priceService.submitBulkUpdate(request);
    }
}

@Service
@RequiredArgsConstructor
public class PriceCommandService {

    private final PriceRuleRepository priceRepo;
    private final KafkaTemplate<String, Object> kafka;
    private final StringRedisTemplate redis;

    @Transactional
    public void updatePrice(PriceUpdateRequest req) {
        // 1. Upsert to PostgreSQL (durable)
        PriceRule rule = PriceRule.builder()
            .productId(req.getProductId())
            .storeId(req.getStoreId())    // null = national
            .priceCents(req.getPriceCents())
            .effectiveFrom(req.getEffectiveAt() != null ? req.getEffectiveAt() : Instant.now())
            .build();

        priceRepo.save(rule);

        // 2. Immediately update Redis (for instant read freshness)
        String storeField = req.getStoreId() != null ? req.getStoreId() : "NATIONAL";
        redis.opsForHash().put("price:" + req.getSkuId(), storeField,
            String.valueOf(req.getPriceCents()));

        // 3. Invalidate assembled product cache
        redis.delete("product:" + req.getProductId() + ":*");  // pattern delete

        // 4. Publish to Kafka (for store sync, ES update, analytics)
        kafka.send("price-events", req.getSkuId(),
            new PriceChangedEvent(
                UUID.randomUUID().toString(),
                req.getSkuId(),
                req.getStoreId(),
                req.getRegionId(),
                req.getOldPriceCents(),
                req.getPriceCents(),
                req.getPromotionId(),
                req.getEffectiveAt() != null ? req.getEffectiveAt() : Instant.now(),
                Instant.now()
            ));
    }

    public BulkUpdateJobResponse submitBulkUpdate(BulkPriceUpdateRequest req) {
        String jobId = UUID.randomUUID().toString();

        // Store job metadata in Redis
        redis.opsForValue().set("bulk_price_job:" + jobId,
            serialize(req), Duration.ofHours(24));

        // Publish a job trigger event — bulk-price-processor service handles it
        kafka.send("bulk-price-jobs", jobId, req);

        return new BulkUpdateJobResponse(jobId, "ACCEPTED",
            "Job submitted. Poll GET /api/v1/prices/bulk/" + jobId + " for status.");
    }
}

// Redis-based price updater (Kafka consumer)
@Component
@RequiredArgsConstructor
public class RedisPriceUpdater {

    private final StringRedisTemplate redis;

    @KafkaListener(topics = "price-events", groupId = "redis-price-updater",
                   concurrency = "10")  // 10 consumer threads
    public void handlePriceEvent(PriceChangedEvent event) {
        String storeField = event.getStoreId() != null ? event.getStoreId() : "NATIONAL";
        redis.opsForHash().put("price:" + event.getSkuId(), storeField,
            String.valueOf(event.getNewPriceCents()));

        // For national price changes, also update all store-specific cache entries
        // (stores that don't have overrides will now see the new national price)
        if (event.getStoreId() == null) {
            // Invalidate assembled product caches in batch
            // Use Lua script for atomic pipeline delete
            redisInvalidateProductCache(event.getSkuId());
        }
    }

    private void redisInvalidateProductCache(String skuId) {
        // Use scan + delete instead of KEYS (KEYS blocks Redis)
        ScanOptions options = ScanOptions.scanOptions()
            .match("product:*" + skuId + "*")
            .count(100)
            .build();
        redis.scan(options).forEachRemaining(key -> redis.delete(key));
    }
}
```

### 10.4 Search Service

```java
@RestController
@RequestMapping("/api/v1/search")
@RequiredArgsConstructor
public class SearchController {

    private final SearchService searchService;

    @GetMapping("/products")
    public SearchResponse searchProducts(@ModelAttribute SearchRequest request) {
        return searchService.search(request);
    }

    @GetMapping("/suggest")
    public SuggestResponse suggest(@RequestParam String q,
                                   @RequestParam(defaultValue = "5") int size) {
        return searchService.suggest(q, size);
    }
}

@Service
@RequiredArgsConstructor
public class SearchService {

    private final ElasticsearchClient esClient;
    private final StringRedisTemplate redis;

    public SearchResponse search(SearchRequest req) {
        // Check suggestion cache
        String cacheKey = "search:result:" + req.toCacheKey();
        String cachedResult = redis.opsForValue().get(cacheKey);
        if (cachedResult != null) return deserialize(cachedResult, SearchResponse.class);

        // Build ES query
        SearchRequest.Builder builder = new SearchRequest.Builder()
            .index("catalog");

        BoolQuery.Builder boolQuery = new BoolQuery.Builder();

        // Full-text must clause
        if (req.getQuery() != null && !req.getQuery().isBlank()) {
            boolQuery.must(m -> m.multiMatch(mm -> mm
                .query(req.getQuery())
                .fields("name^3", "brand^2", "description", "tags^1.5")
                .fuzziness("AUTO")
                .type(TextQueryType.BestFields)
            ));
        }

        // Filters
        boolQuery.filter(f -> f.term(t -> t.field("status").value("ACTIVE")));

        if (req.getBrand() != null) {
            boolQuery.filter(f -> f.term(t -> t.field("brand.keyword").value(req.getBrand())));
        }
        if (req.getCategoryId() != null) {
            boolQuery.filter(f -> f.term(t -> t.field("categoryId").value(req.getCategoryId())));
        }
        if (req.getMinPrice() != null || req.getMaxPrice() != null) {
            boolQuery.filter(f -> f.range(r -> {
                r.field("priceCents");
                if (req.getMinPrice() != null) r.gte(JsonData.of(req.getMinPrice()));
                if (req.getMaxPrice() != null) r.lte(JsonData.of(req.getMaxPrice()));
                return r;
            }));
        }
        if (req.getStoreId() != null) {
            boolQuery.filter(f -> f.term(t -> t.field("availableStoreIds").value(req.getStoreId())));
        }
        if (req.getMinRating() != null) {
            boolQuery.filter(f -> f.range(r -> r.field("averageRating")
                .gte(JsonData.of(req.getMinRating()))));
        }

        // Attribute filters (e.g., size=10, color=red)
        if (req.getAttributes() != null) {
            for (Map.Entry<String, String> attr : req.getAttributes().entrySet()) {
                boolQuery.filter(f -> f.nested(n -> n
                    .path("attributes")
                    .query(q2 -> q2.bool(b2 -> b2
                        .must(m -> m.term(t -> t.field("attributes.key").value(attr.getKey())))
                        .must(m -> m.term(t -> t.field("attributes.value").value(attr.getValue())))
                    ))
                ));
            }
        }

        // Aggregations for facets
        Map<String, Aggregation> aggs = Map.of(
            "by_category", Aggregation.of(a -> a.terms(t -> t.field("categoryId").size(10))),
            "by_brand",    Aggregation.of(a -> a.terms(t -> t.field("brand.keyword").size(20))),
            "price_ranges", Aggregation.of(a -> a.range(r -> r.field("priceCents")
                .ranges(
                    RangeAggregationRange.of(rr -> rr.to("5000")),
                    RangeAggregationRange.of(rr -> rr.from("5000").to("10000")),
                    RangeAggregationRange.of(rr -> rr.from("10000").to("25000")),
                    RangeAggregationRange.of(rr -> rr.from("25000"))
                )
            )),
            "avg_rating", Aggregation.of(a -> a.avg(avg -> avg.field("averageRating")))
        );

        SearchRequest esRequest = builder
            .query(q -> q.bool(boolQuery.build()))
            .aggregations(aggs)
            .sort(s -> s.score(sc -> sc.order(SortOrder.Desc)))
            .from(req.getPage() * req.getSize())
            .size(req.getSize())
            .build();

        var esResponse = esClient.search(esRequest, ProductSearchDocument.class);
        SearchResponse response = toSearchResponse(esResponse, req);

        // Cache search results briefly (popular searches benefit most)
        redis.opsForValue().set(cacheKey, serialize(response), Duration.ofSeconds(30));

        // Track search term popularity
        if (req.getQuery() != null) {
            redis.opsForZSet().incrementScore("search:popular", req.getQuery(), 1);
        }

        return response;
    }
}
```

### 10.5 Multi-Language Support

```java
// Translation resolution with full fallback chain
@Component
public class TranslationResolver {

    // Fallback chain per locale
    private static final Map<String, List<String>> FALLBACK_CHAIN = Map.of(
        "en-US", List.of("en-US", "en"),
        "en-GB", List.of("en-GB", "en-US", "en"),
        "es-MX", List.of("es-MX", "es", "en-US"),
        "es-ES", List.of("es-ES", "es", "en-US"),
        "fr-FR", List.of("fr-FR", "fr", "en-US"),
        "fr-CA", List.of("fr-CA", "fr-FR", "fr", "en-US"),
        "pt-BR", List.of("pt-BR", "pt", "en-US")
    );

    public Translation resolve(Map<String, Translation> translations, String requestedLocale) {
        List<String> chain = FALLBACK_CHAIN.getOrDefault(requestedLocale,
            List.of(requestedLocale, "en-US"));

        for (String locale : chain) {
            Translation t = translations.get(locale);
            if (t != null && t.getName() != null) return t;
        }
        // Absolute fallback: first available translation
        return translations.values().stream().findFirst()
            .orElse(Translation.empty());
    }
}

// API response includes locale metadata
@JsonInclude(JsonInclude.Include.NON_NULL)
record ProductDetailResponse(
    String  productId,
    String  sku,
    long    version,
    String  name,                    // translated
    String  description,             // translated
    String  shortDescription,        // translated
    String  brand,
    String  categoryId,
    String  categoryPath,
    long    priceCents,
    String  formattedPrice,          // locale-formatted, e.g., "$99.99" vs "99,99 €"
    String  currency,
    String  availability,
    Map<String, Object> specifications,
    List<ImageDto>      images,
    List<VariantDto>    variants,
    String  locale,                  // the locale that was actually served
    String  requestedLocale          // the locale that was requested
) {}
```

### 10.6 Search Indexer (Kafka Consumer → Elasticsearch)

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class SearchIndexerConsumer {

    private final ElasticsearchClient esClient;
    private final ProductAttributeRepository mongoRepo;
    private final ProductRepository pgRepo;
    private final BrandRepository brandRepo;
    private final CategoryRepository categoryRepo;

    @KafkaListener(topics = "product-events", groupId = "search-indexer", concurrency = "5")
    public void handleProductEvent(ProductUpdatedEvent event) {
        try {
            switch (event.getEventType()) {
                case "CREATED", "UPDATED" -> indexProduct(event.getProductId());
                case "ARCHIVED"           -> deleteFromIndex(event.getProductId());
                case "DISCONTINUED"       -> updateStatusInIndex(event.getProductId(), "DISCONTINUED");
            }
        } catch (Exception e) {
            log.error("Failed to index product {}: {}", event.getProductId(), e.getMessage());
            // Will be retried by Kafka consumer (at-least-once)
        }
    }

    @KafkaListener(topics = "price-events", groupId = "search-indexer-price", concurrency = "5")
    public void handlePriceEvent(PriceChangedEvent event) {
        // Update only the price field in Elasticsearch (partial update)
        if (event.getStoreId() == null) {
            // National price change — update the searchable base price
            esClient.update(u -> u
                .index("catalog")
                .id(event.getSkuId())
                .doc(Map.of("priceCents", event.getNewPriceCents())),
                Void.class
            );
        }
    }

    private void indexProduct(String productId) throws IOException {
        // Fetch from both sources to build a rich, denormalized search document
        Product core = pgRepo.findById(productId).orElseThrow();
        ProductAttributes attrs = mongoRepo.findByProductId(productId);
        Brand brand = brandRepo.findById(core.getBrandId()).orElseThrow();
        Category category = categoryRepo.findById(core.getCategoryId()).orElseThrow();

        // Flatten attributes for filtering
        List<Map<String, String>> flattenedAttrs = flattenSpecifications(attrs.getSpecifications());

        // Build completions for suggestions
        List<String> suggestions = buildSuggestions(core.getName(), brand.getName(),
            attrs.getTranslations());

        ProductSearchDocument doc = ProductSearchDocument.builder()
            .productId(productId)
            .sku(core.getSku())
            .name(core.getName())
            .description(getEnglishDescription(attrs))
            .brand(brand.getName())
            .brandId(brand.getId())
            .categoryId(core.getCategoryId())
            .categoryPath(category.getPath())
            .categoryName(category.getName())
            .priceCents(core.getBasePriceCents())
            .status(core.getStatus().name())
            .attributes(flattenedAttrs)
            .tags(buildTags(core, attrs))
            .imageUrl(getPrimaryImageUrl(productId))
            .translations(attrs.getTranslations())
            .suggest(suggestions)
            .updatedAt(core.getUpdatedAt())
            .build();

        esClient.index(i -> i
            .index("catalog")
            .id(productId)
            .document(doc)
        );

        log.info("Indexed product {} (SKU: {})", productId, core.getSku());
    }
}
```

---

## 11. Failure Scenarios & Mitigations

### Scenario 1: Redis Cache Failure

```
Problem: Redis down → 10,000 reads/sec → all hit PostgreSQL/MongoDB
         Both databases can handle ~1,000 req/sec → 10x overload → crash

Mitigation:
  1. Redis Sentinel / Redis Cluster (3+ nodes, auto-failover <30s)
  2. Circuit breaker (Resilience4j): if Redis error rate > 10% in 10s,
     open circuit → serve stale from in-process Caffeine L1 cache
  3. Caffeine L1 cache acts as blast shield (30s TTL, survives Redis outage)
  4. Rate limit incoming reads during Redis recovery (shed excess load)
  5. Pre-populate Caffeine before Redis recovery completes

Code:
  @CircuitBreaker(name = "redis-cache", fallbackMethod = "getCachedProductFallback")
  public ProductDetailResponse getFromRedis(String key) { ... }

  public ProductDetailResponse getCachedProductFallback(String key, Exception e) {
      return localCache.getIfPresent(key);  // L1 fallback
  }
```

### Scenario 2: Elasticsearch Down (Search Unavailable)

```
Problem: Elasticsearch cluster unavailable → search returns 503

Mitigation:
  1. Elasticsearch has 3-node cluster with 1 replica per shard
     (single node failure is transparent)
  2. On full cluster failure:
     - Circuit breaker opens → fallback to MongoDB text search
     - MongoDB text index on product name, sku, brand
     - Slower (~500ms) but functional for basic search
  3. Show degraded search banner to users
  4. Search results with 0 facets (MongoDB doesn't provide aggregations like ES)

Degraded mode response includes:
  { "degraded": true, "message": "Advanced filters temporarily unavailable" }
```

### Scenario 3: Price Inconsistency (Redis Price Lost)

```
Problem: Redis evicts/loses price hash → reads fall through to PostgreSQL
         → PostgreSQL has base price, not store-specific promotional price

Mitigation:
  1. Redis with persistence (AOF + RDB snapshots) — price data survives restart
  2. On Redis startup: price-service runs "warm-up" query
     SELECT sku_id, store_id, price_cents FROM current_prices
     → Loads all active prices into Redis pipeline (~10 seconds for 500K SKUs)
  3. Prices in PostgreSQL are always the source of record
     Redis cache is a performance layer, not the only copy
  4. TTL-less price keys: prices don't expire from Redis (only updated by events)
     Avoids accidental expiry of active prices
```

### Scenario 4: Kafka Consumer Lag on Price Updates

```
Problem: Heavy promotion → 10,000 price events/sec → Kafka consumers can't keep up
         Redis prices lag → stores serve stale prices

Mitigation:
  1. Monitor consumer lag (target <1,000 messages lag)
  2. Scale redis-price-updater consumer group: 50 instances × 1 partition each
  3. For urgent "immediate" promotions: price-service writes to Redis FIRST
     (synchronous), then publishes Kafka async for durability/fan-out
  4. Use Redis pipelining for bulk updates:
     Pipeline 10,000 HSET commands in one network round trip → ~100ms
```

### Scenario 5: Database Schema Migration on 500K Products

```
Problem: Adding a new required column to PostgreSQL products table
         ALTER TABLE on 500K rows → table lock → reads fail

Mitigation:
  1. Use online schema migrations (pg_repack or zero-downtime approach)
  2. Expand-contract pattern:
     Phase 1: Add nullable column (no lock if nullable)
     Phase 2: Backfill in batches (1,000 rows at a time, with delays)
     Phase 3: Add NOT NULL constraint + default value
     Phase 4: Remove old column if applicable
  3. Coordinate with application deployments:
     Deploy app that writes both old and new columns
     Run migration
     Deploy app that reads only new column
```

---

## 12. Scaling Strategy

### Service Scaling Targets

```
Service                   Scale Model     Instances    Notes
──────────────────────────────────────────────────────────────────────────
catalog-query-service     Horizontal      20-100       Stateless, scales with traffic
product-service           Horizontal      5-10         Write bottleneck is DB, not app
price-service             Horizontal      5-10         Write path; DB is bottleneck
search-service            Horizontal      10-20        Stateless; ES handles scale
search-indexer            Horizontal      10-20        Kafka consumer group
store-sync-service        Horizontal      10           One per Kafka partition
api-gateway               Horizontal      10-20        L7 routing only

Redis                     Cluster (6)     6 nodes      3 primaries, 3 replicas
PostgreSQL                1P + 3 RR       4 nodes      Reads → replicas, writes → primary
MongoDB                   RS (3 nodes)    3 nodes      Primary + 2 secondaries
Elasticsearch             Cluster (3)     3 nodes      5 shards × 1 replica
Kafka                     Cluster (6)     6 brokers    50 partitions for price-events
```

### Read Path Scaling for 10,000 req/sec

```
Tier 1 — CDN Edge (cache-control: max-age=60)
  Static pages: ~70% of traffic served from edge (product detail pages)
  → Reduces origin load to 3,000 req/sec

Tier 2 — Caffeine L1 Cache (in each catalog-query-service instance)
  5,000 hot products × 30-instance cluster
  Hit ratio: ~80% of remaining requests
  → Reduces Redis load to 600 req/sec per Redis node

Tier 3 — Redis L2 Cache
  95% hit ratio on 10,000 products × 30 min TTL
  → Reduces DB load to <100 req/sec

Tier 4 — PostgreSQL / MongoDB
  <100 req/sec actual DB queries — trivially handled
```

### Write Path Scaling for 100 writes/sec + Price Spikes

```
Product writes (100/sec):
  product-service (5 instances) → PostgreSQL primary
  PostgreSQL write throughput: ~5,000 writes/sec → not a bottleneck

Price writes (up to 10,000/sec):
  price-service → Redis pipeline (immediate)
              → Kafka (fan-out to 50 consumer instances)
  Redis HSET: 100,000 ops/sec per node → not a bottleneck
  Kafka: 500K msgs/sec → not a bottleneck at 50 partitions
```

---

## 13. Monitoring & Observability

### Key Metrics (Prometheus + Grafana)

```
catalog_read_requests_total{service, status}      → req/sec by service
catalog_read_latency_p99{service}                 → target <50ms product detail
catalog_search_latency_p99                        → target <200ms search
catalog_cache_hit_rate{level}                     → L1 / L2 / overall
catalog_cache_miss_total{level}                   → unexpected spikes indicate issues

price_update_events_total                         → price write volume
price_propagation_latency_p99                     → time Redis updated after DB write
elasticsearch_index_lag_seconds                   → time since last ES update

kafka_consumer_lag{group, topic}                  → target <1,000 for price-events
redis_memory_used_bytes                           → watch for OOM
redis_evicted_keys_total                          → should be 0 for price keys
postgresql_connections_active                     → near pool limit = scale DB
mongodb_query_latency_p99                         → target <10ms for attribute fetch
```

### Alerts

```
CRITICAL:
  catalog_read_latency_p99 > 500ms   → Cache layer failing
  redis_evicted_keys{type="price"}   → Price data evicted — inaccurate prices
  price_propagation_latency > 10s    → Price pipeline stuck
  elasticsearch_cluster_status=RED   → Search unavailable

WARNING:
  catalog_cache_hit_rate < 80%       → TTL too short or cache too small
  kafka_consumer_lag > 5000          → Search indexer falling behind
  postgresql_connections > 80%       → Scale connection pool or add replica
  catalog_search_latency_p99 > 500ms → ES cluster undersized
```

---

## 14. Summary Cheat Sheet

### The 5 Core Answers at a Glance

| Question | Answer | Key Technology |
|---|---|---|
| What microservices? | product-service (write), catalog-query-service (read), price-service, search-service, availability-service, category-service, image-service | CQRS split |
| Database structure? | PostgreSQL = core catalog + price rules (ACID); MongoDB = flexible specs + translations; Redis = hot cache + price map; Elasticsearch = search index | Polyglot persistence |
| Real-time price updates? | Write PostgreSQL → immediately update Redis → publish Kafka → propagate to stores. All stores converge in <5s | Redis Hash HSET + Kafka |
| Search functionality? | Elasticsearch with inverted index, BM25 relevance, faceted aggregations, fuzzy matching, synonym filters. Synced via Kafka | Elasticsearch |
| Catalog consistency? | Event versioning + Kafka ordered by SKU + per-store version tracking + reconciliation job for offline stores | Kafka partitioning + Redis version stamps |

### Golden Rules for Product Catalog Systems

```
1. Separate read and write models (CQRS) — 100:1 read ratio demands specialized optimization
2. Redis is the price read layer — never read store prices from PostgreSQL on every request
3. Elasticsearch for search — SQL LIKE queries on 500K products will kill your database
4. MongoDB for flexible product attributes — SQL schemas cannot handle variable product specs
5. CDN for product images — databases are not blob stores; serve binaries from edge
6. Kafka for price propagation — synchronous fan-out to 1,000 stores is too slow and brittle
7. Event versioning prevents stale data — every update carries a monotonic version
8. Partition Kafka by SKU — guarantees price events for the same product are ordered
9. Pre-warm caches before traffic peaks — cold cache at peak = thundering herd on DB
10. Fallback chain for translations: specific locale → language → en-US → first available
11. Two-level cache (Caffeine L1 + Redis L2) for hot products — reduces Redis load by 80%
12. Index aliases for zero-downtime Elasticsearch re-indexing — swap alias, not index name
```

### Technology Role Summary

```
PostgreSQL:        ★ Product core data (sku, name, category, status)
                   ★ Price rules — store/regional/national pricing
                   ★ Category hierarchy (recursive CTEs)
                   ★ Source of record for all writes

MongoDB:           ★ Rich product specifications (schema-flexible per category)
                   ★ Multi-language translations (all locales in one document)
                   ★ Product variants and attribute groups
                   ★ Image metadata

Redis:             ★ Price map (Hash per SKU: field=storeId, value=priceCents)
                   ★ Assembled product cache (L2, 30min TTL)
                   ★ Availability flags per store (short TTL)
                   ★ Category tree cache (24h TTL)
                   ★ Rate limiting for catalog admins

Elasticsearch:     ★ Full-text search with relevance ranking (BM25)
                   ★ Faceted aggregations (category, brand, price range)
                   ★ Fuzzy matching + synonym expansion
                   ★ Search-as-you-type completions

Kafka:             ★ Decouple product writes from search indexing + store sync
                   ★ Ordered price events per SKU (partitioned by skuId)
                   ★ Fan-out to 1,000 stores without blocking writes
                   ★ Replay capability for re-indexing or catching up offline stores

Spring Boot:       ★ Catalog query assembly (multi-source, parallel fetch)
                   ★ Kafka consumers for indexing, price sync, cache invalidation
                   ★ CQRS write commands with optimistic locking
                   ★ Circuit breakers around all external dependencies (Resilience4j)
```
