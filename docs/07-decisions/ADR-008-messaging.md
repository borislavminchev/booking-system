# ADR-008: Messaging — Kafka for the event stream, Redis for SSE fan-out

- **Status:** Accepted
- **Date:** 2026-11-15
- **Decision:** D72 — resolves [ADR-004](ADR-004-transactional-outbox.md) Q1 and [ADR-005](ADR-005-sse-fan-out.md) Q1

## Context

ADR-004 needs a durable, ordered-per-key, replayable event stream fed by outbox relays and consumed by several services (read models, Scheduler, Notifications, Realtime). ADR-005 needs a lightweight pub/sub with many short-lived per-recipient topics (`customer:{id}`, `merchant:{id}`) plus a shared replay buffer of ≥5 min.

## Options

| Option | Pros | Cons |
|---|---|---|
| **Kafka + Redis** | Kafka: durable, partitioned by key, consumer groups, replay, compacted topics for catalog snapshots; Redis: cheap dynamic channels and Streams for replay | Two technologies |
| NATS JetStream only | One tech for both roles | Less common as a managed service; smaller ecosystem |
| Redis Streams only | Simplest stack | Weaker durability/retention for the system-of-record event stream |

## Decision

- **Kafka** (managed, 3 brokers across AZs, RF=3, `min.insync.replicas=2`, producers `acks=all`) is the durable event stream. Topics per producing context (`identity.events`, `catalog.events`, `booking.events`, `wallet.events`, `review.events`, `scheduler.events`), keyed by aggregate id; `catalog.events` is log-compacted so replicas can be rebuilt.
- **Redis Pub/Sub** carries per-recipient SSE topics; **Redis Streams** hold the ≥5 min replay buffer per topic.
- A fan-out component in the Realtime tier consumes Kafka and republishes to Redis.

## Consequences

- Consumers are Kafka consumer groups, idempotent by event id and version (ADR-004).
- Losing Redis only degrades live updates; clients refetch (ADR-005).
- Kafka unavailability delays read models and notifications but not bookings or payments (outbox buffers).
