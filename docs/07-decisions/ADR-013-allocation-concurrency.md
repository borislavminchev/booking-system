# ADR-013: Allocation and concurrency model for the booking core

- **Status:** Accepted
- **Date:** 2026-11-18
- **Decisions:** D86, D87
- **Detail:** [Deep-dive 01 Booking core §3–§8](../06-deep-dives/01-booking-core/01-booking-core.md)

## Context

No resource may be double-booked under any concurrency (NFR-101). Expired holds and deadlines must free capacity immediately, without waiting for a job (NFR-107). Each booking must have exactly one winning transition (NFR-109). And 7 years of history must not slow down the hot path (NFR-405). PostgreSQL exclusion constraints cannot reference `now()`, so "expired" rows can't simply be left out of the constraint.

## Options

| Option | Pros | Cons |
|---|---|---|
| **Live-only allocation table + GiST exclusion + `expires_at` mirror + reap on conflict + history table** | DB-enforced invariant; small hot table and index; effective state without jobs | Reap step in every claim; history table to maintain |
| One allocation table with `active` flag and partial exclusion constraint | Simpler writes | Table/index grow to ~375M rows; expired holds still block until a job clears them |
| Advisory lock per resource + overlap `SELECT` | No constraint maintenance | Correct only if every code path locks; serialises non-overlapping holds |
| Discrete slot rows with unique keys | Simple uniqueness | Doesn't fit arbitrary durations, buffers or multi-day segments |
| SERIALIZABLE isolation for everything | Fewer explicit locks | Retry storms under contention; no gain over constraint + row locks |

## Decision

1. `resource_allocation` holds only live allocations, with `EXCLUDE USING gist (resource_id WITH =, range WITH &&, booking_id WITH <>)`. Released rows move to `allocation_history` (partitioned by month of range start) in the same transaction.
2. `expires_at` on each allocation equals the instant its booking stops holding capacity (hold expiry, `verify_by`, approval deadline; NULL when confirmed). It is only changed by `UPDATE … WHERE expires_at > now()` under a row lock (monotonic expiry rule).
3. Claims first delete expired overlapping rows on their resources ("reap"), then insert, in `resource_id` order.
4. Transactions run in READ COMMITTED with: idempotency row as lock → booking `FOR UPDATE` + version CAS → guest token → allocations → wallets → append-only inserts. The booking clock is the transaction's `now()`, with 2 s timeouts. Writes materialise lagging effective state first. Serialization or deadlock failures are retried in-request.

## Consequences

- No-double-booking is a database invariant, proven in deep-dive 01 §8.1, and re-checked nightly by reconciliation (NFR-110).
- The hot table stays at a few million rows whatever the history size.
- Availability's replica must apply the same `expires_at` rule (deep-dive 03).
- Counted capacity (physical stock) will need a second allocation strategy behind the same interface (NFR-1303).
