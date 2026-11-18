# Non-Functional Requirements

> Status: v1.0 — targets D39–D47 decided 2026-11-11; items marked (D47) are the confirmed operational defaults (see §15).
> Scope comes from [Product Vision & Purpose](../01-product/vision-and-purpose.md) §6 (core qualities); decisions referenced as Dxx live in its decisions log (§9). Functional requirements are in [functional-requirements.md](functional-requirements.md).
> Requirement IDs are stable: never renumber, mark removed ones as *withdrawn*.

## 0. Conventions

- **MUST** = MVP requirement. **SHOULD** = MVP if cheap, otherwise first follow-up.
- Latencies are **server-side** (request received → response sent), measured per operation over a rolling 30 days unless stated otherwise.
- **Booking path** = slot query, hold, checkout, cancel, reschedule, verify, approve/reject.
- **Money state** = wallet balances, reservations and ledger transactions.

## 1. Correctness & consistency

The system's most critical guarantees (Vision §6 qualities 2 and 7). These are never traded for latency or availability: when in doubt, the system refuses the operation rather than risk a wrong result.

| ID | Requirement | Ref |
|---|---|---|
| NFR-101 | **No double-booking:** at no moment may two active holds/bookings overlap on the same resource (including buffers and multi-day spans), under any concurrency of customers, staff and system jobs. | FR-503 |
| NFR-102 | Holds, bookings and money state MUST be **strongly consistent** (linearizable per resource and per wallet). Eventual consistency is allowed only for read models: availability display, search, calendars, ratings, notifications. | |
| NFR-103 | **Atomic checkout:** payment, no-show reservation and booking creation (FR-504) MUST succeed or fail together. No partial outcome is ever visible or persisted. | FR-504 |
| NFR-104 | Every booking state change that moves money (cancel, reject, auto-reject, no-show, complete, reschedule) MUST apply the state change and its money movements atomically. | FR-509, FR-603–605 |
| NFR-105 | **Ledger invariants** hold at all times: double-entry (every transaction sums to zero across wallets); no wallet's available balance below zero; reserved ≤ balance; balances always equal the sum of their ledger entries. | FR-903–905 |
| NFR-106 | All state-changing APIs MUST be **idempotent** via a client-supplied idempotency key; a retried request MUST NOT create a second hold, booking or transaction. | |
| NFR-107 | Time-based transitions (hold expiry, verification cutoff, approval deadline, auto-complete) MUST be **correct regardless of job lag**: an expired hold or deadline is treated as expired by every read/write the moment it passes, even before cleanup runs. | FR-502, FR-704, FR-801, FR-608 |
| NFR-108 | Scheduled transitions MUST be executed **exactly once in effect** (at-least-once execution, idempotent outcome) and survive restarts/failover. Cleanup SHOULD run within 30 s of the due time. (D47) | |
| NFR-109 | Concurrent conflicting actions (e.g. customer cancels while staff marks no-show; approval arrives as the deadline passes) MUST resolve to exactly one winning transition; the loser gets a clear conflict error. | FR-507 |
| NFR-110 | A nightly **reconciliation** job MUST verify NFR-101 and NFR-105 across all data and alert on any violation. | |

## 2. Availability & disaster recovery

| ID | Requirement | Ref |
|---|---|---|
| NFR-201 | Booking path availability: **99.9%** monthly (≈ 43 min downtime/month). | D39 |
| NFR-202 | Other surfaces (marketplace browse/search, back office, reviews, wallet top-up/withdraw) SHOULD meet 99.9%; they MAY degrade independently without affecting the booking path. | D39 |
| NFR-203 | Deployment: **one EU cloud region, at least 3 availability zones**; loss of any single AZ MUST NOT cause data loss or more than a few minutes of disruption. | D40 |
| NFR-204 | **RPO ≈ 0** for bookings, holds and money state: no acknowledged write may be lost (synchronous replication across AZs before acknowledging). | D41 |
| NFR-205 | **RTO ≤ 1 h** for a full-region or data-store disaster, from encrypted backups / cross-region copies. | D41 |
| NFR-206 | Backups: point-in-time recovery for at least 14 days; one copy outside the primary region; restore tested at least quarterly. (D47) | D41 |
| NFR-207 | Deployments and schema migrations MUST be **zero-downtime** (rolling, backward-compatible). | |
| NFR-208 | Graceful degradation: if a dependency fails (notifications, search, SMS/email provider), booking and money operations MUST still work; deferred side effects are queued and delivered later. | |

## 3. Performance (latency)

Targets are p95; **p99 ≤ 2× p95** for all of them.

| ID | Operation | p95 target | Ref |
|---|---|---|---|
| NFR-301 | Slot query (one service, one day, one merchant) | ≤ 300 ms | D42 |
| NFR-302 | Create hold | ≤ 300 ms | D42 |
| NFR-303 | Checkout (pay + reserve + create booking) | ≤ 500 ms | D42 |
| NFR-304 | Cancel / reschedule / approve / reject / verify | ≤ 500 ms | D42 |
| NFR-305 | Marketplace search & browse | ≤ 500 ms | D42 |
| NFR-306 | Other reads (booking lists, calendar, wallet, merchant page) | ≤ 200 ms | D42 |
| NFR-307 | Embeddable widget initial load (client, 4G) SHOULD be interactive in ≤ 2 s. (D47) | |

## 4. Scale & capacity

| ID | Requirement | Ref |
|---|---|---|
| NFR-401 | Design target (D18): ~10k merchants, ~100k bookings/day, peaks of a few hundred bookings/sec. | D18 |
| NFR-402 | Size for **3× peak at launch**: ~1,000 holds+checkouts/sec and ~10,000 slot queries/sec sustained for 15 min, within the §3 latency targets. | D43 |
| NFR-403 | Growth beyond that MUST be possible by **adding instances** (horizontal scale-out), without redesigning the data model. | D43 |
| NFR-404 | **Hot tenants:** a single merchant releasing popular slots (e.g. a restaurant opening a month of tables) MUST NOT degrade other tenants; contention stays local to that merchant's resources. | |
| NFR-405 | Data volume planning: ≥ 7 years of bookings and ledger entries online or queryable (≈ 250M bookings, ≈ 1B+ ledger rows at target scale). | D46 |
| NFR-406 | Load tests at NFR-402 levels MUST pass before launch and before major releases. | |

## 5. Real-time & freshness

| ID | Requirement | Ref |
|---|---|---|
| NFR-501 | Shown availability MUST reflect a new hold/booking/cancellation within **≤ 2 s** (p99). | D44, FR-405 |
| NFR-502 | Staleness MUST never cause a double-booking: the hold is always validated against the authoritative state (NFR-101). A stale slot fails with a clear "just taken" message and fresh alternatives. | FR-503 |
| NFR-503 | Merchant calendars and in-app booking status SHOULD update live (push) within ≤ 2 s. | D44 |
| NFR-504 | Marketplace search index (profile, services, ratings) MAY lag up to 5 min. (D47) | |

## 6. Multi-tenancy & isolation

| ID | Requirement | Ref |
|---|---|---|
| NFR-601 | **Strict tenant isolation:** no merchant user can read or change another merchant's data. Enforced in the data-access layer by default, not left to each endpoint. | Vision §6 (1) |
| NFR-602 | Customers see only their own bookings/wallet across merchants, and merchants see a customer only through bookings with that merchant (FR-607). | D3 |
| NFR-603 | Automated tests MUST cover cross-tenant access for every API (negative tests). | |
| NFR-604 | **Noisy-neighbour protection:** per-tenant and per-client rate limits on APIs and the widget; one tenant cannot exhaust shared capacity. | |
| NFR-605 | Per-tenant configuration and policies are versioned/snapshotted per booking (FR-311). | FR-311 |

## 7. Security

| ID | Requirement | Ref |
|---|---|---|
| NFR-701 | **MFA required for Owners**; optional for Staff and Customers. | D45 |
| NFR-702 | Passwords hashed with a modern adaptive algorithm (e.g. Argon2id/bcrypt); breached-password check on sign-up; login rate limiting and lockout. | |
| NFR-703 | Authorisation by role (Owner/Staff/Customer/Guest) checked server-side on every request; least privilege (Staff cannot touch config, policies, wallet — FR-204). | D28 |
| NFR-704 | Guest links (verify, cancel) and SMS codes: single-purpose, unguessable (≥ 128-bit tokens), expiring, rate-limited; codes limited to a few attempts. | FR-702, FR-706 |
| NFR-705 | TLS 1.2+ for all traffic; encryption at rest for all data stores and backups; secrets in a managed secret store. | |
| NFR-706 | Protection against OWASP Top 10; the embeddable widget MUST be safe to embed on third-party sites (no merchant site can act as the customer beyond the booking flow). | FR-105 |
| NFR-707 | Bot/abuse protection on hold creation and guest booking (holds are free and lock capacity): per-client/IP limits on concurrent holds; CAPTCHA or challenge when abuse is detected. | FR-502 |
| NFR-708 | Wallet operations (withdrawals, large changes) are logged with actor and context; withdrawals SHOULD require recent re-authentication. | FR-902 |

## 8. Privacy & data retention

| ID | Requirement | Ref |
|---|---|---|
| NFR-801 | **GDPR is the privacy baseline for all users**, regardless of country. | D40 |
| NFR-802 | All personal data stored and processed in the EU region (NFR-203); sub-processors (email, SMS, push) under data-processing agreements. | D40 |
| NFR-803 | **Ledger transactions, booking history and audit trail retained 7 years.** | D46 |
| NFR-804 | **Right to erasure:** on account deletion, personal data (name, email, phone, review text) is erased or pseudonymised; ledger and booking records remain with a pseudonymous reference so money invariants (NFR-105) still hold. | D46 |
| NFR-805 | Guest personal data (name, email, phone) is kept only as long as needed for the booking and its retention period; data export available to customers on request (portability). | D46 |
| NFR-806 | Personal data MUST NOT appear in logs, metrics or analytics in clear text. | |

## 9. Auditability

| ID | Requirement | Ref |
|---|---|---|
| NFR-901 | Every booking state transition records time (UTC), actor (user, guest, or System + job) and reason; records are append-only. | FR-507 |
| NFR-902 | Ledger transactions are **immutable**; corrections are compensating transactions, never edits or deletes. | FR-905 |
| NFR-903 | Configuration and policy changes (services, prices, policies, roles) are logged with before/after values and actor. | FR-311 |
| NFR-904 | From the audit trail it MUST be possible to explain any wallet balance and any booking's full history. | Vision §6 (7) |

## 10. Time & time zones

| ID | Requirement | Ref |
|---|---|---|
| NFR-1001 | Slots are defined in the merchant's local time zone (IANA zone id); instants stored in UTC alongside the zone. | D10 |
| NFR-1002 | DST transitions MUST be handled correctly: no non-existent slots in spring-forward gaps, no duplicated slots in fall-back overlaps; multi-day bookings across a DST change keep correct opening-hour accounting. | D30 |
| NFR-1003 | Time-zone database updates MUST be applied without code changes; existing bookings keep their local wall-clock time. | |
| NFR-1004 | Server clocks synchronised (NTP); deadlines evaluated against server time, never client time. | NFR-107 |

## 11. Notifications delivery

| ID | Requirement | Ref |
|---|---|---|
| NFR-1101 | Notifications are **at-least-once** with de-duplication (a recipient never gets the same message twice in normal operation). | FR-1002–1003 |
| NFR-1102 | Transactional notifications (confirmation, verification, approval, cancellation) dispatched within **≤ 60 s** of the event (p95). (D47) | |
| NFR-1103 | Guest verification email/SMS dispatched within **≤ 30 s** (p95), since the verification window can be short (D31). (D47) | D31 |
| NFR-1104 | Reminders sent within **± 5 min** of the configured offset. (D47) | FR-1004 |
| NFR-1105 | Provider failures are retried with back-off; persistent failures are visible to operations and never block the booking (NFR-208). | |

## 12. Observability & operability

| ID | Requirement | Ref |
|---|---|---|
| NFR-1201 | Metrics, structured logs and distributed traces for all services, with request and tenant correlation ids. | |
| NFR-1202 | SLOs for §2, §3 and §5 dashboarded with error-budget alerting. | |
| NFR-1203 | Business-invariant alerts: double-booking detected, ledger imbalance, scheduled-transition backlog, notification backlog. | NFR-110 |
| NFR-1204 | Infrastructure as code; reproducible environments (dev, staging, prod). | |

## 13. Maintainability & extensibility

| ID | Requirement | Ref |
|---|---|---|
| NFR-1301 | **API-first:** web, mobile and widget use the same versioned public APIs; breaking changes only in a new version. | Vision §6 (10) |
| NFR-1302 | The money boundary is an interface so a real payment provider can replace the virtual wallet without changing booking logic. | D7, FR-907 |
| NFR-1303 | The domain model MUST allow the listed future features (chains, physical stock, chained services, table combining, platform fee, multi-language, availability search) to be added without rewriting the booking core. | Vision §8 |
| NFR-1304 | New verticals are added by configuration (resource types, services), not code per vertical. | Vision §5 |

## 14. Usability & clients

| ID | Requirement | Ref |
|---|---|---|
| NFR-1401 | Customer-facing web and widget meet **WCAG 2.1 AA**. (D47) | |
| NFR-1402 | Supported clients: last 2 versions of major browsers (Chrome, Safari, Firefox, Edge), responsive down to 360 px width; mobile apps on current and previous major iOS/Android. (D47) | |
| NFR-1403 | All user-facing text is externalised (single language now, multi-language later — D10). | D10 |

## 15. Open questions

| # | Question | Status | Raised |
|---|---|---|---|
| Q1 | Confirm the proposed defaults: scheduled-transition lag ≤ 30 s (NFR-108), backup retention (NFR-206), widget load (NFR-307), search-index lag (NFR-504), notification timings (NFR-1102–1104), accessibility (NFR-1401), supported clients (NFR-1402) | Resolved → D47 | 2026-11-11 |
