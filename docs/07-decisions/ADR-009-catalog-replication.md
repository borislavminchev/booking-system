# ADR-009: Catalog data on the booking path — local replicas fed by events

- **Status:** Accepted
- **Date:** 2026-11-15
- **Decision:** D73

## Context

Holds, checkouts and slot computation need merchant configuration owned by Merchant Catalog: services and their versions, resource requirements, resources, opening/working hours, exceptions and policy. The booking path must meet p95 ≤300/500 ms and 99.9% availability without depending on Catalog being up (NFR-201, NFR-208).

## Options

| Option | Pros | Cons |
|---|---|---|
| **Local replicas via events** | No runtime dependency or extra hop on the booking path; replicas can be indexed for slot math | Config lag of ~seconds; replica rebuild tooling needed |
| Synchronous call to Catalog (with cache) | Always fresh | Adds latency and couples booking availability to Catalog |
| Catalog inside the booking service | No replication | Larger core service mixing config UI traffic with the hot path |

## Decision

- Catalog writes every config change with an outbox row and publishes a **full, versioned snapshot** of the changed entity on `catalog.events` (compacted, keyed by merchant id).
- Booking & Payments and Availability keep their own replica tables, applied idempotently by version.
- A booking's terms snapshot records the service and policy versions read from the replica (FR-311).

## Consequences

- A configuration change becomes effective for new bookings after replication lag (typically < 1 s). A booking made in that window uses the previous version, which FR-311's snapshot rule allows. Pausing (FR-104) behaves the same way.
- Replicas can be rebuilt by replaying the compacted topic; replication lag is monitored and alerted.
- Catalog outages affect only configuration and public profile, not booking.
