# Booking System — Project Context

Read this file first in every session. Keep it current: update **Current status** and **Next steps** at the end of each work block.

## What this project is

We're designing (and later building) a **multi-tenant booking platform** for local service businesses: salons, barbershops, restaurants, auto repair shops and wellness clinics. Merchants publish bookable capacity (staff, tables, rooms, bays, equipment). Customers discover, hold, pay for and confirm bookings in real time, through each merchant's booking page/widget or a cross-merchant marketplace.

Full product definition: [docs/01-product/vision-and-purpose.md](docs/01-product/vision-and-purpose.md). Its **decisions log (§9)** is the source of truth for scope.

### Key MVP decisions (summary — see the decisions log for detail)

- Marketplace across all merchants, plus a per-merchant booking page/widget
- One global customer account; registration required for any payment; guests can make payment-free bookings only
- Guest verification: merchant toggles it on/off and sets a cutoff; unverified bookings are auto-cancelled at the cutoff
- Virtual wallets (virtual deposit/withdraw, single currency); deposits, prepayments, no-show fees
- No-show fee reserved in the customer's wallet at booking; cancellation policy defined per merchant
- Manual approval per service: the pending booking holds the slot and is charged; refunded on reject or auto-reject
- Multiple countries (time-zone aware), single language, no platform commission
- Merchants self sign-up; fixed list of categories; no platform-operator tooling
- Design target: ~10k merchants, ~100k bookings/day, peaks of a few hundred bookings/sec; sized for 3× peak
- NFRs: 99.9% availability, one EU region multi-AZ, GDPR baseline, RPO≈0/RTO≤1h, checkout p95 ≤500 ms, availability fresh ≤2 s, MFA for Owners, 7y ledger/audit retention
- One service per booking (may need several resources at once); multi-day bookings; single restaurant tables
- 10-min platform-wide checkout hold; reschedule only within the free-cancellation window
- Roles: Owner + Staff; staff-created bookings carry no payments
- Notifications: email + in-app/push; SMS only for guest verification
- Reviews & ratings in MVP; search by availability is future
- Late guest bookings use a shorter late-booking cutoff; unmarked bookings auto-complete 24h after end
- Domain (D48–D63): hold = Booking in state Held, abandoned → Expired; reschedule swaps allocations on the same booking; separate CustomerAccount / MerchantUser identities; guest/walk-in contact embedded on the booking, client history by email/phone; double-entry ledger with Available/Reserved (customer) and Available/Pending (merchant) accounts; one upfront payment (deposit or prepayment); remainder settled at venue or via wallet (reserved at checkout); no-show keeps upfront + captures fee; approval deadline = min(request + window, start − margin); cancelling PendingApproval = full refund; PendingApproval can be rescheduled (new deadline); Completed/NoShow terminal; MerchantUser email globally unique
- API (D64–D67): REST/JSON + SSE for live updates; one `/v1` API with zones `/auth`, public `/merchants`, `/bookings`, `/me`, `/manage/merchants/{merchant_id}`; bearer JWT access + rotating refresh, guest booking token + single-purpose link tokens; POST action endpoints per transition, Idempotency-Key on every POST, version/ETag, RFC 9457 errors, cursor pagination, cross-tenant → 404
- Events (D68–D69): transactional outbox + relay (at-least-once, idempotent consumers); SSE fan-out via per-user pub/sub topics, shared replay store, no sticky sessions
- Architecture (D70–D75): microservices behind one API gateway (Identity, Merchant Catalog, Booking & Payments, Availability, Discovery/Search, Reviews, Notifications, Scheduler, Realtime/SSE); Booking + Ledger in one service (one ACID tx per transition); PostgreSQL per service + Redis; Kafka event stream + Redis SSE fan-out; catalog replicated via events; effective state + Scheduler for timed transitions; Kubernetes, one EU region × 3 AZs, cloud-agnostic
- Requirements check (D76–D79): push device tokens in Notifications, "in-app" = live booking lists (no inbox in MVP); staff find registered customers by exact email/phone only; staff↔resource link stored on the Resource (Catalog); reminder offsets snapshotted per booking (changes apply to new bookings)
- Identity (D80): self-hosted Keycloak (realms `customers`, `merchants`) for registration, login, MFA, tokens via OIDC auth code + PKCE; thin Identity service for merchant sign-up, invitations, profile/deletion, customer lookup and `identity.events`
- Booking core (D81–D89): single PostgreSQL primary now, shard-ready (cell id bits in ids), merchant cells + customer wallet cells later (ADR-012); "any available" = least busy that day, tables smallest fit; reschedule: customer may keep current resource; staff bookings/moves within hours unless logged override, no min_notice; guest verify_by capped at start − approval_margin for approval services; live-only allocation table + GiST exclusion, expires_at mirrors effective deadline, reap on conflict, history table (ADR-013); READ COMMITTED + booking row lock + CAS + global lock order; terms snapshotted at hold; monthly partitions by UUIDv7
- Future: in-app notification inbox, re-applying changed reminder offsets to existing bookings, physical stock, chains/branches, real payments, multi-language, waitlists/recurring/group bookings, operator tooling

## How we work

- **Ask when something is unclear**; never assume. Ask clarifying questions as **pick-lists** (multiple-choice picker), with a recommended option first.
- Every product or design decision goes into a **decisions log** with an ID (D1, D2, …) and a date. Architecture and API decisions become ADRs in `docs/07-decisions/`.
- Unresolved items are marked **[Open]** and listed in the document's open-questions section.
- Docs are Markdown under `docs/`, grouped by phase (see `docs/README.md`). Add every new doc to the index in `docs/README.md`.
- Diagrams are drawn in **draw.io**: keep the `.drawio` source and the exported `.png` side by side (e.g. `docs/05-architecture/diagrams/`) and reference the PNG from the Markdown.
- Markdown hygiene: no blank lines inside tables; one table per log.

## Docs layout

| Folder | Purpose |
|---|---|
| `docs/01-product/` | Vision, purpose, users, qualities, scope, decisions |
| `docs/02-requirements/` | Functional and non-functional requirements |
| `docs/03-domain/` | Entities, relationships, glossary, domain rules |
| `docs/04-api-design/` | Client-facing API design: styles per use case, conventions, endpoint/channel catalogue |
| `docs/05-architecture/` | High-level architecture |
| `docs/06-deep-dives/` | Per-feature designs |
| `docs/07-decisions/` | ADRs |

## Design phases

1. **Product** — vision, users, scope, decisions log.
2. **Requirements** — functional (FR) and non-functional (NFR) requirements.
3. **Domain** — entities, relationships, glossary, domain rules (state machines, ledger model).
4. **API design** — client-facing API style per interaction (REST, gRPC, GraphQL, WebSockets, SSE, webhooks, …), conventions, endpoint catalogue.
5. **Architecture** — high-level architecture: components, service boundaries, internal APIs/events, data stores, tenancy enforcement, scheduled transitions.
6. **Deep-dives** — per-feature designs.
7. **Implementation.**

## Current status

- **Phase:** 6 — Deep-dives: 01 Booking core ✅ (v1.0); next 02 Wallet & ledger
- Phase 1 ✅ vision doc v1.6, decisions D1–D89
- `docs/02-requirements/`: ✅ FR v1.2, NFR v1.0
- `docs/03-domain/`: ✅ domain-model v1.2; booking-lifecycle v1.2 (draw.io state diagram in `03-domain/diagrams/`); ledger-model, glossary v1.0
- `docs/04-api-design/`: ✅ api-conventions.md v0.3, endpoint-catalogue.md v0.4 (login via Keycloak OIDC; keep_resource, override_hours)
- `docs/05-architecture/`: ✅ high-level-architecture.md v0.4 (incl. FR requirements check, §10.2) + 4 draw.io diagrams (container, event flow, hold/checkout sequence, deployment) in `diagrams/`
- `docs/07-decisions/`: ADR-001–005 (API, outbox, SSE), ADR-006 (service decomposition), ADR-007 (data stores), ADR-008 (messaging), ADR-009 (catalog replication), ADR-010 (scheduled transitions), ADR-011 (Keycloak identity provider), ADR-012 (booking scale-out), ADR-013 (allocation and concurrency model)
- `docs/06-deep-dives/`: backlog of 18 (README.md v0.2); ✅ `01-booking-core/01-booking-core.md` v1.0 + 4 sequence diagrams (hold, checkout, reschedule, move) in `01-booking-core/diagrams/`
- Open architecture questions (architecture doc §12), each assigned to a deep-dive: internal sync protocol + service auth (Q1), RLS for tenancy (Q2), guest-token delivery (Q3), slot computation strategy (Q5), concrete cloud (Q6), Keycloak details (Q7). Q4 resolved (D81)
- Hand-offs from 01: merchant-wallet credit contention → 02; shared segment/slot library and `expires_at` rule in Availability → 03; timer commands + sweep → 04; outbox table → 05
- **Last updated:** 2026-11-18 (deep-dive 01 Booking core done)

## Next steps

1. **Deep-dives** (phase 6) — backlog with scope, requirements, outputs and done criteria in [docs/06-deep-dives/README.md](docs/06-deep-dives/README.md); one subfolder per deep-dive `docs/06-deep-dives/NN-<slug>/` with the doc `NN-<slug>.md` and its own `diagrams/` subfolder. Tick sub-steps off as each deep-dive is completed.
   - [ ] **Core**
     - [x] 01 Booking core (Q4 → D81)
     - [ ] 02 Wallet & ledger
     - [ ] 03 Availability & slot computation (Q5)
     - [ ] 04 Scheduled transitions & Scheduler
     - [ ] 05 Event platform & data replication
     - [ ] 06 Realtime / SSE
     - [ ] 07 Guest verification (Q3)
     - [ ] 08 Notifications
     - [ ] 09 Identity / Keycloak (Q7)
     - [ ] 10 Tenancy enforcement (Q2)
     - [ ] 11 API gateway, rate limiting & abuse protection
   - [ ] **Cross-cutting**
     - [ ] 12 Security architecture (Q1)
     - [ ] 13 Privacy, retention & audit
     - [ ] 14 Capacity, performance & load testing
     - [ ] 15 Resilience, HA & DR
     - [ ] 16 Observability & operations
   - [ ] **Optional**
     - [ ] 17 Platform & deployment (Q6)
     - [ ] 18 Clients & widget
     - [ ] API refinements when needed: OpenAPI spec for core endpoints, request/response schemas, SSE event catalogue
2. **Implementation** (phase 7) — to be detailed once the deep-dives are done.
