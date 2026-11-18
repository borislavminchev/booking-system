# ADR-012: Booking & Payments scale-out — single primary now, merchant cells later

- **Status:** Accepted
- **Date:** 2026-11-18
- **Decision:** D81
- **Resolves:** architecture Q4
- **Detail:** [Deep-dive 01 Booking core §10](../06-deep-dives/01-booking-core/01-booking-core.md#10-scale-out-path-q4--adr-012-d81)

## Context

Booking & Payments applies state, allocations and money in one local ACID transaction (ADR-006). At 3× peak (NFR-402) it needs ~3,000 short write transactions/s. A single PostgreSQL primary sustains roughly 10–15k such transactions/s, so it has 3–5× headroom. NFR-403 still requires growth by adding instances without redesigning the data model. The natural shard keys conflict: bookings, allocations and merchant wallets are keyed by **merchant**, but the customer wallet debited at checkout is keyed by **customer**.

## Options

| Option | Pros | Cons |
|---|---|---|
| **Single primary now, shard-ready; merchant cells + customer wallet cells later** | Simplest launch with full ACID; no cross-shard logic until it is needed; clear path | Cross-cell checkout needs a two-step authorization when activated |
| Merchant shards from day one | Proven scale from launch | Saga and cell routing built and operated before any load needs them |
| Distributed SQL (CockroachDB / YugabyteDB) | Keeps one ACID transaction across shards | No GiST exclusion constraints (overlap enforcement must be redesigned); consensus write latency; new operational stack |
| One global wallet DB + merchant shards | Customer wallets in one place | Still cross-database at checkout; wallet DB becomes the next bottleneck |

## Decision

1. **Launch on one primary** (dedicated cluster, synchronous standby).
2. **Shard-ready from day one:** `merchant_id` on every booking-side row; 8 bits of every booking/wallet UUIDv7 reserved for a cell id (0 today); DB access through a `CellRouter`; no cross-merchant joins on command paths.
3. **Scale-out shape when needed:** *merchant cells* (bookings, allocations, events, merchant wallets and their ledger accounts) keyed by merchant through a directory; *wallet cells* (customer wallets) keyed by `hash(customer_id)`.
4. **Cross-cell money:** checkout = synchronous two-step (authorize on wallet cell → commit booking + merchant entries on merchant cell → void on failure; recovery job for stuck authorizations). Credits (refunds, releases, captures) = async idempotent transfers via the outbox. Inter-cell clearing accounts must net to in-flight transfers (reconciliation).
5. **Triggers** (sustained over a week of peaks): primary CPU > 60%, sync-commit p95 > 10 ms, checkout DB time p95 > 100 ms, or projected load > 50% of measured capacity within 6 months. Read offload to replicas and vertical scaling come first.

## Consequences

- Launch keeps the simplest correct design; all no-double-booking guarantees stay in one database.
- When cells are activated, a customer may briefly see a "pending checkout" authorization before the booking confirms. This is accepted then, and documented against NFR-103.
- Merchants can be moved between cells with a short per-merchant write freeze; the directory overrides id-encoded cells.
- Load tests (deep-dive 14) must measure the real primary limit, to calibrate the triggers.
