# Booking System — Documentation

Living documentation for the multi-tenant booking platform. Documents are grouped by design phase and updated as the system evolves.

## Structure

| Folder | Purpose | Status |
|---|---|---|
| `01-product/` | Product vision, purpose, user types, core qualities | Done (v1.6) |
| `02-requirements/` | Functional and non-functional requirements | Done (FR v1.2, NFR v1.0) |
| `03-domain/` | Entities, relationships, glossary, domain rules | Done (domain model v1.2, booking lifecycle v1.2, others v1.0) |
| `04-api-design/` | Client-facing API design: styles per use case, conventions, endpoint/channel catalogue | First pass (conventions v0.3, catalogue v0.4) |
| `05-architecture/` | High-level architecture, components, data flow (draw.io sources + PNGs in `diagrams/`) | First pass (v0.4) |
| `06-deep-dives/` | Per-feature and cross-cutting designs (booking core, availability, events, security, resilience, …) | 01 Booking core done; 17 to go |
| `07-decisions/` | Architecture and API Decision Records (ADRs) | ADR-001–013 |

## Index

- [Product Vision & Purpose](01-product/vision-and-purpose.md) — purpose, users, qualities, scope, decisions log
- [Functional Requirements](02-requirements/functional-requirements.md) — FRs by area, v1.2
- [Non-Functional Requirements](02-requirements/non-functional-requirements.md) — consistency, availability, latency, scale, isolation, security, privacy, audit; v1.0
- [Domain Model](03-domain/domain-model.md) — modules, entities, relationships, ER diagram; v1.2
- [Booking Lifecycle](03-domain/booking-lifecycle.md) — booking state machine, transitions, time rules, reschedule, concurrency; state diagram in [03-domain/diagrams](03-domain/diagrams/); v1.2
- [Ledger Model](03-domain/ledger-model.md) — wallets, double-entry accounts, money flows, invariants; v1.0
- [Glossary](03-domain/glossary.md) — ubiquitous language; v1.0
- [API Conventions](04-api-design/api-conventions.md) — styles, URL zones, formats, idempotency, concurrency, errors, auth, SSE; v0.3
- [Endpoint Catalogue](04-api-design/endpoint-catalogue.md) — endpoints by zone mapped to FRs and lifecycle transitions; v0.4
- [High-Level Architecture](05-architecture/high-level-architecture.md) — services, data stores, events, key flows, deployment, FR coverage + requirements check; diagrams in [05-architecture/diagrams](05-architecture/diagrams/); v0.4
- [Deep-Dives Backlog](06-deep-dives/README.md) — 18 deep-dives with scope, FRs/NFRs, open questions, outputs and done criteria; v0.2
- [01 Booking core](06-deep-dives/01-booking-core/01-booking-core.md) — schema, transaction per transition T1–T18, allocation/overlap model, any-of pick, reschedule, partitioning, scale-out; diagrams in [01-booking-core/diagrams](06-deep-dives/01-booking-core/diagrams/); v1.0
- [ADR-001 API style](07-decisions/ADR-001-api-style.md) — REST + SSE
- [ADR-002 API surface](07-decisions/ADR-002-api-surface.md) — one API, scoped zones
- [ADR-003 API authentication](07-decisions/ADR-003-api-authentication.md) — bearer tokens + guest tokens (tokens issued by Keycloak, ADR-011)
- [ADR-004 Transactional outbox](07-decisions/ADR-004-transactional-outbox.md) — reliable event publishing
- [ADR-005 SSE fan-out](07-decisions/ADR-005-sse-fan-out.md) — per-user pub/sub topics across servers
- [ADR-006 Service decomposition](07-decisions/ADR-006-service-decomposition.md) — microservices; Booking & Payments as one service
- [ADR-007 Data stores](07-decisions/ADR-007-data-stores.md) — PostgreSQL per service, Redis, search on PostgreSQL
- [ADR-008 Messaging](07-decisions/ADR-008-messaging.md) — Kafka event stream, Redis SSE fan-out
- [ADR-009 Catalog replication](07-decisions/ADR-009-catalog-replication.md) — local replicas fed by catalog events
- [ADR-010 Scheduled transitions](07-decisions/ADR-010-scheduled-transitions.md) — effective state + Scheduler service
- [ADR-011 Identity provider](07-decisions/ADR-011-identity-provider.md) — self-hosted Keycloak, thin Identity service
- [ADR-012 Booking scale-out](07-decisions/ADR-012-booking-scale-out.md) — single primary now, merchant cells later
- [ADR-013 Allocation and concurrency](07-decisions/ADR-013-allocation-concurrency.md) — live allocations + exclusion constraint, transaction conventions
