# ADR-006: Service decomposition — microservices, with Booking and Ledger in one service

- **Status:** Accepted
- **Date:** 2026-11-15
- **Decision:** D70

## Context

The domain has clear modules ([domain model §2](../03-domain/domain-model.md#2-modules-candidate-bounded-contexts)) with different consistency, load and change profiles: the booking path is strongly consistent and hot; discovery, reviews and notifications are eventually consistent and can degrade independently (NFR-202, NFR-208). At the same time every money-moving transition must change booking state and ledger atomically (NFR-103, NFR-104), and the ledger model leaves the cross-aggregate atomicity mechanism to architecture.

## Options

| Option | Pros | Cons |
|---|---|---|
| Modular monolith | Simplest operations; local transactions everywhere | One deployable; weaker isolation of failures and scaling |
| **Microservices, Booking + Ledger together** | Independent scaling/deploys per context; the only cross-context atomicity need stays a local transaction | Booking & Payments is the largest service |
| Microservices, Booking and Wallet split (TCC saga) | Purest split | Try/confirm/cancel flows, timeouts and reconciliation on the hottest path; extra latency against the 500 ms checkout target |

## Decision

Microservices behind one API gateway, each with its own database:

Identity · Merchant Catalog · **Booking & Payments** (Booking module + Ledger module) · Availability · Discovery/Search · Reviews · Notifications · Scheduler · Realtime/SSE gateway.

- Booking & Payments keeps the two modules separate in code (own packages, own tables, the ledger only reachable through its module API and the payment-provider interface), so a later split stays possible.
- Services integrate through events (ADR-004, ADR-008); synchronous internal calls are limited to the few listed in the [architecture §5.3](../05-architecture/high-level-architecture.md#53-synchronous-service-to-service-calls).

## Consequences

- Every lifecycle transition with money is one ACID transaction; no saga on the booking path.
- Read-heavy and non-critical surfaces (slots, search, reviews, notifications, SSE) scale and fail independently of the booking core.
- More operational surface than a monolith: CI/CD per service, tracing, contract tests for events.
- Booking & Payments is the scaling hot spot; its scale-out path is an open deep-dive question.
