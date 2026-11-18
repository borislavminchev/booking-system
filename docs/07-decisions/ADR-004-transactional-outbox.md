# ADR-004: Reliable event publishing — transactional outbox + relay

- **Status:** Accepted (broker: Kafka, see [ADR-008](ADR-008-messaging.md))
- **Date:** 2026-11-12
- **Decision:** D68

## 1. Context

Booking and wallet changes must trigger work outside the transaction: SSE pushes (NFR-503), notifications (FR-1001–1004, NFR-1101), and read models such as availability, calendars, search and ratings (NFR-501, NFR-504). Booking state and money change atomically in the database (NFR-103, NFR-104). Publishing straight to a message broker from the request handler is a **dual write**:

- the DB commits, then the process crashes before publishing → the change happened but nobody is told;
- the publish succeeds, then the DB rolls back → everyone is told about a change that never happened.

The same applies to scheduled transitions (hold expiry, verification cutoff, approval deadline, auto-complete), which also have to emit events (NFR-108).

## 2. Options

| Option | Pros | Cons |
|---|---|---|
| **Transactional outbox + relay** | Event committed atomically with the change; publishing retried until it succeeds; no distributed transaction | Extra table and worker; at-least-once delivery means duplicates |
| Publish after commit, best effort | Simplest | Lost events on crash; phantom events on rollback |
| Distributed transaction (2PC: DB + broker) | Atomic in theory | Poor broker support, slow, fragile; not worth it |
| Event sourcing (events are the source of truth) | Events are native | Large shift in the whole data model for one need |

## 3. Decision

1. Every state change that others must learn about writes an **outbox row in the same DB transaction** as the change itself (booking state, allocations, ledger). This holds for user actions and scheduled jobs alike.
2. Outbox row: `id` (monotonic), `aggregate_type`, `aggregate_id`, `event_type`, `routing` (merchant_id, customer_id, assigned staff user ids), `payload` (JSON), `created_at`, `published_at`.
3. An **outbox relay** worker reads unpublished rows in `id` order, publishes them to the broker, then marks them published.
   - Start with a **polling publisher** (`… WHERE published_at IS NULL ORDER BY id LIMIT n FOR UPDATE SKIP LOCKED`, poll interval ≤ 200 ms).
   - Change data capture (reading the DB's change log) is an allowed later swap if poll load or latency requires it; producers don't change.
4. **Delivery is at-least-once.** Every consumer MUST be idempotent: deduplicate by event `id`, and ignore events whose `version` is not newer than what it already has.
5. **Ordering:** events are published keyed by `booking_id` (or `merchant_id`), so events of one booking stay in order on the broker; consumers still guard with `version`.
6. Published rows are deleted after 7 days. The outbox is not the audit trail — `BookingEvent` and the ledger are (NFR-901, D46).

## 4. Event envelope

```json
{
  "id": "01J9Z3…",
  "type": "booking.updated",
  "occurred_at": "2026-11-20T07:12:03Z",
  "aggregate": {"type": "booking", "id": "0193…", "version": 3},
  "routing": {"merchant_id": "…", "customer_id": "…", "staff_user_ids": ["…"]},
  "data": {"state": "confirmed", "previous_state": "pending_approval"}
}
```

Internal consumers (notifications, read models) may receive richer `data`; what reaches clients over SSE is the thin form defined in [API conventions §9](../04-api-design/api-conventions.md#9-real-time-updates-sse).

## 5. Consequences

- No lost or phantom events, without distributed transactions.
- Event latency = commit → relay poll → broker → consumer; with ≤ 200 ms polling this fits the ≤ 2 s targets (NFR-501, NFR-503) and notification targets (D47).
- Relay must be monitored: outbox backlog size and age of the oldest unpublished row are key alerts.
- Several relay instances can run concurrently (`SKIP LOCKED`); per-booking order then relies on keyed publishing plus `version` checks.

## 6. Open questions

| # | Question | Status | Raised |
|---|---|---|---|
| Q1 | Broker technology for the durable event stream (e.g. Kafka, RabbitMQ, NATS JetStream, Redis Streams) | Resolved → D72 (Kafka, ADR-008) | 2026-11-12 |
