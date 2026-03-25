# System Design Deep Dives

Detailed system design documents for e-commerce backend scenarios using **Java 17, Spring Boot 3, PostgreSQL, MongoDB, Redis, Kafka**, and related technologies.

## Design Documents

| # | Topic | File |
|---|-------|------|
| 01 | Product Catalog Management System | [01-product-catalog-management-system.md](./01-product-catalog-management-system.md) |
| 02 | Shopping Cart System | [02-shopping-cart-system.md](./02-shopping-cart-system.md) |
| 03 | Order Processing Pipeline | [03-order-processing-pipeline.md](./03-order-processing-pipeline.md) |
| 04 | Flash Sale System | [04-flash-sale-system-design.md](./04-flash-sale-system-design.md) |
| 05 | Inventory Management Across Multiple Warehouses | [05-inventory-management-across-multiple-warehouses.md](./05-inventory-management-across-multiple-warehouses.md) |

## Tech Stack

- **Java 17** + **Spring Boot 3**
- **PostgreSQL** — ACID transactional core
- **MongoDB** — Flexible document store (audit logs, product attributes)
- **Redis** — Caching, distributed locks, atomic Lua scripts
- **Kafka** — Event streaming, Saga orchestration, Outbox relay
- **Elasticsearch** — Full-text search, faceted filtering
- **Resilience4j** — Circuit breakers, retries, bulkheads

## Key Patterns Covered

- Saga Pattern (Orchestration & Choreography)
- Transactional Outbox Pattern
- CQRS (Command Query Responsibility Segregation)
- Two-Phase Inventory Reservation
- Idempotency (3-layer payment protection)
- Redis Lua atomic scripts
- Distributed Locks (Redisson / SETNX)
- Dead Letter Queue (DLQ)
- Cache Stampede Prevention (L1 Caffeine + L2 Redis)
- Event Sourcing / Immutable Audit Logs
- Warehouse Selection Scoring Algorithm
- Three-Level Inventory Model (on_hand / on_hold / available)
- Reconciliation with Auto-Approve Thresholds
- CDC (Debezium) for analytics pipeline sync
