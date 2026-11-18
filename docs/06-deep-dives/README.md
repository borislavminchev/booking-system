# Deep-Dives — Backlog

> Status: v0.2 (2026-11-18) — backlog defined; done: [01 Booking core](01-booking-core/01-booking-core.md).
> Deep-dives refine the [high-level architecture](../05-architecture/high-level-architecture.md) one area at a time: they make it cleaner, more scalable and more performant, and resolve its open questions (§12, Q1–Q7). Together they must cover every NFR section in the [non-functional requirements](../02-requirements/non-functional-requirements.md).

## 1. How a deep-dive is written

- One subfolder per deep-dive, `NN-<slug>/`, containing the doc `NN-<slug>.md` (e.g. `01-booking-core/01-booking-core.md`); add it to the index in [docs/README.md](../README.md).
- Diagrams in draw.io: `.drawio` source + exported `.png` (scale 2×) side by side in the deep-dive's own `diagrams/` subfolder (e.g. `01-booking-core/diagrams/`), referenced from the doc.
- Suggested sections: Context & scope · Requirements addressed (FR/NFR ids) · Design (components, data model, flows, diagrams) · Failure modes & edge cases · Scaling & performance · Alternatives considered · Decisions (Dxx, ADRs in `07-decisions/`) · Open questions · Change log.
- Every decision goes into the decisions log (Vision §9) with an ID and date; architecture choices become ADRs. Anything that changes the high-level architecture is reflected there (doc + diagrams).
- A deep-dive is **done** when its "done when" criteria below are met, its open questions are resolved or explicitly deferred, and CLAUDE.md is updated.

## 2. Backlog

| # | Deep-dive | Main NFR areas | Resolves | Priority |
|---|---|---|---|---|
| 01 | [Booking core](01-booking-core/01-booking-core.md) ✅ | §1 correctness, §4 scale, §13 extensibility | Q4 | Core |
| 02 | Wallet & ledger | §1 correctness, §9 audit | — | Core |
| 03 | Availability & slot computation | §3 performance, §5 freshness, §10 time | Q5 | Core |
| 04 | Scheduled transitions & Scheduler | §1 (107–108) | — | Core |
| 05 | Event platform & data replication | §5 freshness, §2 degradation, §13 evolution | — | Core |
| 06 | Realtime / SSE | §5 (503) | — | Core |
| 07 | Guest verification | §7 (704) | Q3 | Core |
| 08 | Notifications | §11 | — | Core |
| 09 | Identity (Keycloak) | §7 (701–702, 708) | Q7 | Core |
| 10 | Tenancy enforcement | §6 (601–603) | Q2 | Core |
| 11 | API gateway, rate limiting & abuse protection | §6 (604), §7 (706–707) | — | Core |
| 12 | Security architecture | §7 | Q1 | Cross-cutting |
| 13 | Privacy, retention & audit | §8, §9 | — | Cross-cutting |
| 14 | Capacity, performance & load testing | §3, §4 | — | Cross-cutting |
| 15 | Resilience, HA & disaster recovery | §2 | — | Cross-cutting |
| 16 | Observability & operations | §12 | — | Cross-cutting |
| 17 | Platform & deployment | §2 (203, 207), §12 (1204) | Q6 | Optional |
| 18 | Clients & widget | §3 (307), §14 | — | Optional |

Suggested order: 01 → 18. Cross-cutting deep-dives (12–16) may run alongside the core ones once 01–05 are done, because they build on their designs.

## 3. Deep-dive definitions

### 01 Booking core

- **Scope:** Booking & Payments booking module — Held/checkout transactions, allocation model (`tstzrange` + GiST exclusion constraint, effective expiry), "any of" candidate selection, multi-day segments, reschedule tentative sets, move/extend, compare-and-set, idempotency table, BookingEvent, partitioning, scale-out path beyond one primary.
- **Requirements:** FR-501–511, FR-601–608, FR-304–305; NFR-101–106, NFR-109, NFR-302–304, NFR-401–405, NFR-1303–1304 (future features and new verticals without rewriting the core).
- **Resolves:** Q4 (sharding: merchant-keyed bookings vs customer-keyed wallets, cross-shard checkout).
- **Outputs:** schema + transaction design per transition, sequence diagrams (hold, checkout, reschedule, move), concurrency analysis, scale-out ADR.
- **Done when:** every lifecycle transition T1–T18 has a defined transaction; no-double-booking is proven under concurrent holds, reschedules and expiries; a scale-out path is decided.

### 02 Wallet & ledger

- **Scope:** ledger module — account/transaction/entry schema, wallet locking and hot-wallet contention, cached balances, M1–M5 and W1–W2 implementation, nightly reconciliation (invariants 1–7), payment-provider interface, ledger partitioning for 7 years, re-auth for withdrawals.
- **Requirements:** FR-901–907; NFR-105, NFR-110, NFR-708, NFR-902, NFR-904, NFR-1302.
- **Outputs:** schema, posting rules, reconciliation job design, provider interface contract.
- **Done when:** every money flow is specified as entries, invariants have checks and alerts, and the provider interface is defined.

### 03 Availability & slot computation

- **Scope:** slot algorithm (hours ∩ working hours ± exceptions − effective allocations, buffers, multi-resource, party size, multi-day), guest and approval rules, cache layout and invalidation, read-model replicas, time zones and DST.
- **Requirements:** FR-401–406, FR-708; NFR-301, NFR-402, NFR-501–502, NFR-1001–1004.
- **Resolves:** Q5 (on-demand + cache vs precomputed).
- **Outputs:** algorithm spec with DST cases, cache/invalidation design, freshness budget.
- **Done when:** 10k slot queries/s at p95 ≤300 ms and ≤2 s freshness are shown to be achievable; DST edge cases are covered.

### 04 Scheduled transitions & Scheduler

- **Scope:** timer model, polling with `SKIP LOCKED`, sharding of timers, internal command API, retries, safety sweep, reminders (snapshot offsets, D79), lag monitoring, rebuild of timers from booking state.
- **Requirements:** FR-502, FR-608, FR-704, FR-801, FR-1004; NFR-107–109, NFR-1104.
- **Outputs:** timer schema, command contracts, failure/race analysis (e.g. approval vs deadline).
- **Done when:** ≤30 s lag at peak is shown, races resolve to one winner, and lost timers are always caught.

### 05 Event platform & data replication

- **Scope:** outbox table and relay (polling vs CDC), Kafka topics/partitions/retention/compaction, event envelope and schema versioning (registry, compatibility rules), consumer idempotency and ordering, replica rebuild from compacted topics (ADR-009), backlog alerts, Kafka outage behaviour.
- **Requirements:** NFR-208, NFR-501, NFR-1101, NFR-1301; ADR-004, ADR-008, ADR-009.
- **Outputs:** topic catalogue, event schema catalogue, versioning rules, replay/rebuild runbook.
- **Done when:** every event type has a schema and owner, and replica rebuild and outage recovery are specified.

### 06 Realtime / SSE

- **Scope:** fan-out component, Redis channels and Streams, subscribe/unsubscribe lifecycle, role filtering, `Last-Event-ID` replay and `event: reset`, heartbeats, connection limits and scaling of SSE pods, client refetch strategy (version skip, debounce, ETag), end-to-end flow diagram with concrete technologies.
- **Requirements:** FR-601, FR-802; NFR-503; ADR-005, ADR-008.
- **Outputs:** end-to-end draw.io sequence (connect → change → push → refetch → reconnect), capacity estimate for connections.
- **Done when:** ≤2 s delivery at peak and reconnect/replay behaviour are specified.

### 07 Guest verification

- **Scope:** GuestToken lifecycle, email link + SMS code, late-booking cutoff, attempts and rate limits, secure delivery of secrets to Notifications, guest cancel link.
- **Requirements:** FR-701–708; NFR-704, NFR-1103.
- **Resolves:** Q3.
- **Outputs:** token design, delivery flow, abuse limits.
- **Done when:** secrets never travel in clear on Kafka, and the ≤30 s dispatch target is met.

### 08 Notifications

- **Scope:** event → template mapping, recipients (Owner, assigned Staff, customer, guest), channels (email, push devices, SMS), preferences and opt-out, dedup key, retries/back-off, provider failover, display snapshot from booking events.
- **Requirements:** FR-1001–1005, FR-802; NFR-1101–1105, NFR-208.
- **Outputs:** notification catalogue, delivery pipeline design.
- **Done when:** every notification in FR-1002/1003 has a trigger, recipients, template and channel.

### 09 Identity (Keycloak)

- **Scope:** realms and clients, merchant membership (Organizations vs attributes), token claims and mappers, Owner MFA flow, refresh rotation settings, re-auth (`max_age`), breached-password extension, event listener → Identity service, reconciliation, theming of login pages and widget popup, Keycloak operations.
- **Requirements:** FR-101, FR-201–202, FR-1301; NFR-701–703, NFR-708; ADR-003, ADR-011.
- **Resolves:** Q7.
- **Outputs:** realm/client configuration, login flows, identity event flow.
- **Done when:** every auth requirement maps to Keycloak configuration or our thin service.

### 10 Tenancy enforcement

- **Scope:** tenant context propagation, data-access layer filters, PostgreSQL row-level security as a second guard, cross-tenant negative tests, opaque ids and `404` rules, client-history keys.
- **Requirements:** NFR-601–603, NFR-605, FR-607.
- **Resolves:** Q2.
- **Outputs:** enforcement design per layer, test strategy.
- **Done when:** a tenant leak would require more than one independent failure, and tests cover every endpoint.

### 11 API gateway, rate limiting & abuse protection

- **Scope:** gateway choice and routing, JWT validation, tenant path check, rate-limit algorithms and keys (caller, tenant, IP), concurrent-hold caps, bot detection and CAPTCHA, widget iframe embedding rules (CSP, frame-ancestors), CDN caching rules.
- **Requirements:** NFR-604, NFR-706–707, NFR-305, NFR-501.
- **Outputs:** gateway configuration model, limit tables, abuse playbook.
- **Done when:** hold flooding and noisy tenants are contained without hurting normal traffic.

### 12 Security architecture

- **Scope:** threat model (STRIDE) of the whole platform, service-to-service protocol and authentication (mTLS, service tokens), network policies, secrets and key management, encryption at rest and in transit, dependency and container security, security testing.
- **Requirements:** NFR-703, NFR-705–706.
- **Resolves:** Q1.
- **Outputs:** threat model, internal protocol/auth ADR, security controls checklist.
- **Done when:** every threat has a control, and Q1 is decided.

### 13 Privacy, retention & audit

- **Scope:** personal-data inventory per service, erasure/pseudonymisation flow across services and Kafka, 7-year retention vs guest data minimisation, data export (portability), no personal data in logs, audit trail (BookingEvent, ledger, config change log), explaining any balance or booking history.
- **Requirements:** NFR-801–806, NFR-901–904; FR-1303.
- **Outputs:** data inventory, retention schedule, erasure flow diagram.
- **Done when:** every personal-data field has an owner, retention rule and erasure path.

### 14 Capacity, performance & load testing

- **Scope:** load model from the design target (3× peak), per-service capacity and DB sizing, caching strategy, partitioning, hot-tenant behaviour, latency budgets per request path, load-test plan and acceptance criteria.
- **Requirements:** NFR-301–307, NFR-401–406, NFR-404.
- **Outputs:** capacity model, latency budgets, load-test plan.
- **Done when:** every p95 target has a budget and every NFR-402 load level has a test scenario.

### 15 Resilience, HA & disaster recovery

- **Scope:** failure modes per component, AZ loss, database failover (RPO≈0), region loss and restore (RTO ≤1 h), backups and quarterly restore drills, graceful degradation matrix, zero-downtime deploys and migrations, chaos testing.
- **Requirements:** NFR-201–208.
- **Outputs:** failure-mode table, DR runbook, degradation matrix.
- **Done when:** every component failure has a defined impact and recovery, and RPO/RTO are demonstrably met.

### 16 Observability & operations

- **Scope:** SLIs/SLOs and error budgets, metrics/logs/traces with tenant and request correlation (including across Kafka), business-invariant alerts, dashboards, on-call runbooks.
- **Requirements:** NFR-1201–1203, NFR-110, NFR-806.
- **Outputs:** SLO catalogue, alert catalogue, dashboard list.
- **Done when:** every SLO and invariant in the NFRs has a signal and an alert.

### 17 Platform & deployment (optional)

- **Scope:** cloud choice and managed-service mapping, Kubernetes layout, environments (dev/staging/prod), infrastructure as code, CI/CD, configuration and feature flags, cost estimate.
- **Requirements:** NFR-203, NFR-207, NFR-1204; D75.
- **Resolves:** Q6.
- **Outputs:** cloud mapping table, environment and pipeline design.
- **Done when:** the target cloud is chosen and every component has a managed or self-hosted mapping.

### 18 Clients & widget (optional)

- **Scope:** web/mobile/back-office client architecture, widget iframe and embed snippet, popup login with Keycloak, SSE client and refetch behaviour, offline/poor-network handling, accessibility, load-time budget.
- **Requirements:** FR-105; NFR-307, NFR-706, NFR-1401–1403.
- **Outputs:** client architecture, widget embedding guide.
- **Done when:** the widget meets the ≤2 s interactive target and WCAG 2.1 AA.

## 4. Change log

| Version | Date | Change |
|---|---|---|
| v0.1 | 2026-11-18 | Backlog of 18 deep-dives mapped to NFR areas and open questions |
| v0.2 | 2026-11-18 | 01 Booking core done (D81–D89, ADR-012, ADR-013); one subfolder per deep-dive with its own `diagrams/` |
