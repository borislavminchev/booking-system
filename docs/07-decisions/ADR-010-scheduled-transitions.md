# ADR-010: Scheduled transitions — effective state + Scheduler service

- **Status:** Accepted
- **Date:** 2026-11-15
- **Decision:** D74

## Context

Hold expiry (10 min), verification cutoff, approval deadline, auto-complete (end + 24 h) and reminders are time-driven (FR-502, FR-608, FR-704, FR-801, FR-1004). They must be correct regardless of job lag (NFR-107), exactly-once in effect and survive restarts (NFR-108), and race cleanly with user actions (NFR-109). Money-moving ones (auto-reject, auto-complete) must apply state and ledger atomically, which only Booking & Payments can do.

## Options

| Option | Pros | Cons |
|---|---|---|
| **Effective state + separate Scheduler service calling Booking & Payments** | Correctness independent of timers; timer engine reusable for reminders; scales separately | One more service; internal command API |
| Polling sweeps inside Booking & Payments only | No extra service | Heavy periodic scans of large tables; reminders still need a scheduler |
| Delayed messages on the broker | No timer store | Kafka has no native delayed delivery; cancellation of timers is awkward |

## Decision

1. **Effective state** in Booking & Payments: every read and write evaluates deadlines against server time; expired allocations never count in the overlap check.
2. **Scheduler** consumes `booking.events`, upserts one durable timer per (booking, deadline type) in its own PostgreSQL, and polls due timers with `FOR UPDATE SKIP LOCKED` (several instances).
3. At due time it calls an **internal, idempotent command** on Booking & Payments (`expire`, `auto-cancel`, `auto-reject`, `auto-complete`), applied by the same compare-and-set as user actions; retried until acknowledged. Target lag ≤30 s.
4. Reminders: the Scheduler emits `reminder.due` on `scheduler.events`; Notifications checks state and opt-outs at send time.
5. A low-frequency **safety sweep** in Booking & Payments materialises overdue bookings whose timer was lost.

## Consequences

- Correctness never depends on the Scheduler; it only keeps materialised state and money movements timely.
- Timer backlog and overdue count are key alerts (NFR-1203).
- Timers are derived data: they can be rebuilt from booking state.
