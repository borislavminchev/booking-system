# Product Vision & Purpose

> Status: v1.6 — initial description plus the product decisions in §9 (D19–D38 added while writing functional requirements, D39–D47 while writing non-functional requirements, D48–D63 while writing the domain model, D64–D69 while designing the API, D70–D75 in the high-level architecture, D76–D79 from the architecture requirements check, D80 identity provider, D81–D89 from deep-dive 01 Booking core).

## 1. One-line purpose

A multi-tenant platform that lets local service businesses **publish and control their bookable capacity**, and lets customers **discover, hold, pay for, and confirm** that capacity in real time — across all businesses on the platform, without phone calls and without double-bookings.

## 2. The problem we solve

| For | Today's pain | What we replace it with |
|---|---|---|
| Merchants | Phone/paper/spreadsheet scheduling, double-bookings, no-shows, staff time spent on calls | One operational engine: schedules, resources, client history, automated comms, deposits & no-show fees |
| Customers | Must call during business hours, wait for callbacks, uncertain confirmation, hard to compare businesses | Browse across businesses, self-service booking, instant confirmation, reschedule/cancel anytime, reminders |

## 3. Three surfaces of the platform

1. **Merchant operations engine** (back office) — the source of truth for *what can be booked, when, with which resources, and under which policies* (deposits, cancellation, verification).
2. **Merchant booking page / embeddable widget** — each merchant's own branded entry point, usable on their website or shared as a link.
3. **Customer marketplace** (web + mobile) — customers browse and search across all businesses and categories, compare, and book with the one they prefer.

All three run on the same **availability & booking engine**: it turns merchant configuration into bookable slots and guarantees a slot is sold at most once. The **virtual wallet** (see §6.3) and notifications are shared supporting capabilities.

## 4. User types

| User | Goal | Key value |
|---|---|---|
| **Merchant owner / admin** | Run the business, configure offering & policies, see performance | Visibility, capacity utilization, fewer no-shows (deposits, fees, verification) |
| **Staff member** | Know their day, manage their own appointments | Personal schedule, client context at hand |
| **Registered customer** | Find and book the right business, pay if required | One account across all merchants; history, payments, self-service changes |
| **Guest customer** | Book quickly without creating an account | No sign-up; limited to bookings that need no payment; must verify the booking (see §6.2) |
| **Platform operator** | Run and moderate the platform | **Future** — not in the MVP (see §8) |

## 5. Target verticals and what differs between them

All verticals book "capacity in time", but they model that capacity differently. Making one model cover all of them is the main generalization challenge.

| Vertical | What is booked | Constrained by | Duration |
|---|---|---|---|
| Hair salon / barbershop | A service with a specific (or any) staff member | Staff, sometimes chair/station | Fixed per service, can chain services |
| Restaurant | A table for a party | Tables (capacity, combinable), covers per interval | Fixed turn time, varies by party size |
| Auto repair shop | A job / drop-off slot | Bays, mechanics, lifts | Variable, often estimated, can span days |
| Wellness clinic | Session with a practitioner | Practitioner **and** room/equipment | Fixed per treatment, buffers for cleanup |

**Takeaway:** the domain model needs a generic notion of *resources* (people, tables, rooms, bays, equipment). A *service* then requires one or more resources for a duration. This avoids writing separate code for each vertical.

## 6. Core qualities (what the system must *be*)

These seed the non-functional requirements.

1. **Multi-tenant** — many independent merchants on shared infrastructure, with strict data isolation and per-tenant configuration and policies.
2. **Correct under concurrency** — no double-booking of a resource, even when several customers and staff act at the same time. This is the system's most critical guarantee.
3. **Real-time** — availability shown to customers reflects the current state; confirmation is instant once the booking's conditions (payment, verification, merchant approval where required) are met.
4. **Hold-then-confirm flow** — a slot is held temporarily (with expiry) while the customer completes checkout, payment, or verification.
5. **Self-service** — customers book, reschedule, cancel without merchant involvement, within merchant-defined policies.
6. **Discoverable** — fast search and browsing across all merchants and categories (by category, location, availability, etc.).
7. **Financially reliable** — wallet balances, deposits, prepayments and no-show fees must stay consistent with booking state: no charge without a booking, no confirmed booking without its required payment, correct refunds, and no money created or lost. Every balance change is auditable.
8. **Multi-country** — merchants and customers in several countries, so time zones are handled correctly (each slot is anchored to the merchant's local time zone). Single virtual currency and single language at launch.
9. **Automated communication** — confirmations, verification requests, approval decisions, reminders, change notifications.
10. **Multi-channel** — web, mobile and embeddable widget on the same APIs.
11. **Operational visibility** — merchants see schedules, resource utilization, and client history.

### 6.1 Customer identity model

- **One global customer account** shared across all merchants.
- **Registration is required** for anything involving payment (deposit, prepayment, no-show fees), since these need a wallet.
- **Guest booking** is allowed only for bookings that require no payment.

### 6.2 Guest booking verification

- Each merchant decides **whether guest bookings need verification** (on/off) and sets the **verification cutoff** (e.g. 10 min before the start time).
- When verification is on, a guest booking is **provisional**. The guest verifies it themselves via a link sent by email or a code sent by SMS (SMS is used only for this — D22).
- If the booking is still unverified at the cutoff, it is **automatically cancelled and the slot released**.
- A guest booking made inside the cutoff uses the merchant's shorter **late-booking cutoff**; the guest is told it expires unless verified in time, and merchant users can verify it themselves (D31).
- Example: a guest books 15:00 with a 10‑minute cutoff. If the booking isn't verified by 14:50, it is cancelled.

### 6.3 Virtual money (MVP)

- All money flows **through the platform** using a **virtual currency** — no real payment provider in the MVP.
- Every registered customer and every merchant has a **wallet** with a balance tracked by the platform.
- Users can **deposit and withdraw** virtual funds. No real money moves.
- Booking payments (deposit, prepayment, no-show fee) move funds between wallets: customer → merchant, or back to the customer on refund.
- Single currency, so no currency conversion.
- The design should keep a clean boundary so a real payment provider can replace the virtual one later.

### 6.4 Merchant approval

- Besides guest verification, a merchant can require **manual approval**, configured **per service**. Staff must accept or reject the booking before it is confirmed.
- A booking pending approval **holds the slot**, and any required payment is **taken immediately**.
- If the booking is rejected, the payment is **fully refunded** and the slot released.
- If staff don't decide by the **approval deadline**, the booking is **auto-rejected** and refunded.
- **Registered customers' bookings are confirmed instantly** (no extra verification). Only required payment or merchant approval can delay confirmation.

### 6.5 Cancellation, refunds & no-show fees

- Each **merchant** defines one cancellation policy: a **free-cancellation window** (e.g. up to 24h before the start). After the window, the deposit or prepayment is kept, fully or in part.
- **No-show fee** amounts are **reserved (frozen) in the customer's wallet at booking time**. The reserved amount is released when the customer attends or cancels within policy, and captured by the merchant on a no-show.
- A booking that needs a reservation can't be made if the customer's available balance doesn't cover it.
- A service has at most one upfront payment: a deposit (partial) or a prepayment (full price) (D56). On a no-show the merchant keeps all of it and captures the no-show fee (D58).
- The remainder of the price is settled per service either at the venue or via the wallet; via wallet, it is reserved at booking and captured when the booking is completed (D57).
- Upfront payments stay in the merchant's non-withdrawable **Pending** balance until a full refund is no longer possible, so refunds are always funded (D55).

### 6.6 Platform revenue

- **No commission** on booking payments in the MVP. The design should allow a platform fee to be added later.

## 7. Core concepts (preliminary glossary)

> Superseded by the domain [glossary](../03-domain/glossary.md); kept here for history.

| Term | Meaning (draft) |
|---|---|
| Tenant / Merchant | A business using the platform; data isolation boundary |
| Category | Business type used for browsing (salon, restaurant, auto repair, clinic, …) |
| Resource | Anything with limited capacity in time: staff, table, room, bay, equipment |
| Service / Offering | What a customer books; needs resources for a duration; carries payment policy |
| Schedule / Availability | When resources are working/open, minus exceptions |
| Slot | A concrete bookable time window derived from availability |
| Hold | Temporary, expiring lock on a slot during checkout |
| Booking / Reservation | A claim on resources for a time window; can be provisional or confirmed |
| Provisional booking | Guest booking awaiting verification; cancelled automatically at the cutoff |
| Customer account | Global registered identity, shared across merchants |
| Guest | Unregistered customer; payment-free bookings only |
| Wallet | Platform-tracked virtual balance of a customer or merchant |
| Transaction | A single, auditable movement of virtual funds (top-up, withdrawal, payment, refund, fee) |
| Deposit / Prepayment | Partial or full payment taken at booking time |
| Pending approval | Booking awaiting a merchant's accept/reject decision; auto-rejected at the approval deadline |
| Reserved funds | Part of a wallet balance frozen for a possible no-show fee; not spendable until released |
| Available balance | Wallet balance minus reserved funds |
| Cancellation policy | Merchant-defined free-cancellation window and the share of payment kept after it |
| No-show fee | Charge applied when the customer doesn't attend, per merchant policy |
| Verification cutoff | Merchant-defined time before start by which a provisional booking must be verified |

## 8. Scope

**In scope (MVP)**

- Merchant back office, merchant booking page/widget, cross-merchant marketplace
- Single-location merchants, in multiple countries (time-zone aware)
- Bookable time-based capacity (staff, tables, rooms, bays, equipment)
- Registered + guest customers, guest verification, optional merchant approval
- Virtual wallets (deposit/withdraw), deposits, prepayments, no-show fees
- Single virtual currency, single language
- Booking notifications and reminders (email + in-app/push; SMS only for guest verification)
- Reviews & ratings by registered customers after a completed booking
- Multi-day bookings (e.g. a car kept in a repair bay)
- Merchant self sign-up; merchants go live in the marketplace without operator review
- Fixed, predefined set of marketplace categories (no operator tooling to manage them)
- Designed for medium scale: ~10k merchants, ~100k bookings/day, peaks of a few hundred bookings/sec

**Future / nice to have**

- Platform operator tooling: merchant approval, category management, moderation/suspension, support and audited wallet adjustments

- Booking physical stock (parts, products) as capacity
- Multi-location merchants / chains (shared staff across branches)
- Real payment provider replacing the virtual wallet
- Multiple languages
- Waitlists, recurring bookings, group bookings / classes
- Several chained services in one booking
- Combining restaurant tables for large parties
- Marketplace search by availability ("free at date/time")
- In-app notification inbox (D76)
- Apply changed reminder offsets to existing bookings' future reminders (D79)

**Out of scope (assumption)**

- Full POS / invoicing
- Payroll and staff HR
- Marketing campaigns beyond booking-related notifications

## 9. Decisions log

| # | Decision | Date |
|---|---|---|
| D1 | Each merchant has its own booking page/widget **and** customers can browse across all businesses and categories (marketplace) | 2026-11-11 |
| D2 | Deposits, prepayments and no-show fees are in scope | 2026-11-11 |
| D3 | One global customer account; registration required for any payment | 2026-11-11 |
| D4 | Guest bookings allowed for payment-free bookings; merchant toggles verification and sets the cutoff; unverified bookings are released at the cutoff | 2026-11-11 |
| D5 | Physical stock capacity → future | 2026-11-11 |
| D6 | Multi-location / chains → future | 2026-11-11 |
| D7 | MVP money flows through the platform as a virtual currency; wallets with virtual deposit/withdraw; no real payments | 2026-11-11 |
| D8 | Merchants can require manual approval of bookings | 2026-11-11 |
| D9 | Waitlists, recurring and group bookings → future | 2026-11-11 |
| D10 | Multiple countries supported; single virtual currency; single language (multi-language → future) | 2026-11-11 |
| D11 | Manual approval per service; pending booking holds the slot and is charged immediately; full refund on reject; auto-reject at the approval deadline | 2026-11-11 |
| D12 | Registered customers' bookings confirmed instantly (no extra verification) | 2026-11-11 |
| D13 | Cancellation policy defined per merchant (free-cancellation window, then deposit/prepayment kept) | 2026-11-11 |
| D14 | No-show fee reserved in the customer's wallet at booking; released or captured afterwards | 2026-11-11 |
| D15 | No platform commission in the MVP | 2026-11-11 |
| D16 | Platform operator role and tooling → future | 2026-11-11 |
| D17 | Merchants self sign-up and go live without review; marketplace categories are a fixed, predefined list | 2026-11-11 |
| D18 | Design target: medium scale (~10k merchants, ~100k bookings/day, peaks of a few hundred bookings/sec) | 2026-11-11 |
| D19 | One service per booking; a service may need several resources at once (e.g. practitioner + room); chained services → future | 2026-11-11 |
| D20 | Reschedule allowed within the free-cancellation window (payments carry over); after it, customer must cancel under policy | 2026-11-11 |
| D21 | Merchant users can create bookings for registered customers or walk-in/phone clients; no wallet payments on these | 2026-11-11 |
| D22 | Notification channels: email + in-app/push; SMS only for guest verification codes | 2026-11-11 |
| D23 | Checkout hold fixed platform-wide at 10 minutes | 2026-11-11 |
| D24 | Restaurants: single table per booking; table combining → future | 2026-11-11 |
| D25 | Multi-day bookings supported (e.g. auto repair) | 2026-11-11 |
| D26 | Reviews & ratings in MVP: registered customers, one per completed booking, 1–5 stars + optional text, public merchant reply, no moderation | 2026-11-11 |
| D27 | Marketplace search by availability → future | 2026-11-11 |
| D28 | Merchant roles: Owner (everything) and Staff (own schedule and bookings) | 2026-11-11 |
| D29 | Deposit, prepayment and no-show fee set per service as a fixed amount or % of price | 2026-11-11 |
| D30 | Multi-day services count opening hours only; resource reserved during open hours on each day spanned | 2026-11-11 |
| D31 | Late guest bookings (inside the verification cutoff) use a shorter merchant-set late-booking cutoff; guest told it expires unless verified; Owner/Staff can verify it | 2026-11-11 |
| D32 | Bookings not marked Completed/No-show are auto-completed 24h after end; reservation released | 2026-11-11 |
| D33 | Pending approvals decided by the Owner or the assigned Staff member | 2026-11-11 |
| D34 | Guests can cancel via email link but not reschedule | 2026-11-11 |
| D35 | Reminder offsets configurable per merchant | 2026-11-11 |
| D36 | Guest bookings are not attached to an account registered later with the same email | 2026-11-11 |
| D37 | Owner/Staff can manually verify any provisional guest booking | 2026-11-11 |
| D38 | With verification on, guests can't book slots starting inside the late-booking cutoff | 2026-11-11 |
| D39 | Availability target 99.9%/month for the booking path | 2026-11-11 |
| D40 | Single EU cloud region, ≥3 availability zones; GDPR is the privacy baseline for all users | 2026-11-11 |
| D41 | DR: RPO ≈ 0 for bookings and money state (sync replication across AZs), RTO ≤ 1h | 2026-11-11 |
| D42 | Latency p95: slot query/hold ≤300 ms, checkout and booking changes ≤500 ms, search ≤500 ms, other reads ≤200 ms; p99 ≤ 2× p95 | 2026-11-11 |
| D43 | Size for 3× the D18 peak at launch (~1,000 bookings/s, ~10,000 slot queries/s); grow by horizontal scale-out | 2026-11-11 |
| D44 | Shown availability fresh within ≤2 s; checkout always re-validates against authoritative state | 2026-11-11 |
| D45 | MFA required for Owners; optional for Staff and Customers | 2026-11-11 |
| D46 | Ledger, booking history and audit kept 7 years; personal data erasable/pseudonymised on account deletion | 2026-11-11 |
| D47 | Operational defaults: scheduled transitions ≤30 s lag; PITR backups 14 days, quarterly restore test; widget interactive ≤2 s; search index lag ≤5 min; transactional notifications ≤60 s, guest verification ≤30 s, reminders ±5 min; WCAG 2.1 AA; last 2 browser versions | 2026-11-11 |
| D48 | A checkout hold is a Booking in state **Held** (one aggregate and id across the whole flow) | 2026-11-12 |
| D49 | A Held booking not checked out within 10 min (or abandoned) ends in terminal state **Expired**, allocations released; Expired rows purged after 30 days | 2026-11-12 |
| D50 | Reschedule keeps the same booking: a tentative allocation set is held for 10 min, then swapped atomically with the current one; payments and reservations carry over | 2026-11-12 |
| D51 | Separate identities: **CustomerAccount** (global) and **MerchantUser** (Owner/Staff, belongs to exactly one merchant); same email may exist in both | 2026-11-12 |
| D52 | Bookings reference the global CustomerAccount directly; guest and walk-in contact details are embedded on the booking (no merchant-scoped client entity) | 2026-11-12 |
| D53 | Merchant client history includes guest and walk-in bookings, grouped by normalised email or phone within the merchant; walk-in bookings require a name and at least one of email/phone | 2026-11-12 |
| D54 | Ledger is double-entry: customer wallet = Available + Reserved accounts, merchant wallet = Available + Pending accounts; reserve/release/capture are ledger transfers; a system Virtual-funding account is the counterpart of top-ups and withdrawals | 2026-11-12 |
| D55 | Upfront payments go to the merchant's **Pending** account (visible, not withdrawable) and move to Available only when a full refund is no longer possible: Completed, No-show, or the kept share of a late cancellation | 2026-11-12 |
| D56 | A service has at most one upfront payment: **deposit** (partial) or **prepayment** (full price), fixed amount or % of price; the no-show fee is independent | 2026-11-12 |
| D57 | Remainder (price − upfront) settled per service either **at venue** or **via wallet**. Via wallet: reserved at checkout, captured on Completed (manual or auto), released on cancel/reject/no-show; counts as a payment term (guests can't book it); staff-created bookings always settle at venue | 2026-11-12 |
| D58 | On a no-show the merchant keeps 100% of the upfront payment and captures the no-show fee; any remainder reservation is released | 2026-11-12 |
| D59 | Approval deadline = min(requested_at + decision window, start − approval margin); window and margin set per service; approval-required slots whose start − margin has passed are not offered | 2026-11-12 |
| D60 | Cancelling a PendingApproval booking is always fully refunded (the cancellation policy applies only once confirmed) | 2026-11-12 |
| D61 | Customers can reschedule Confirmed and PendingApproval bookings (within the free-cancellation window); rescheduling a pending booking keeps it pending and restarts its approval deadline for the new slot | 2026-11-12 |
| D62 | Completed and NoShow are terminal in the MVP; no merchant correction (comes with operator tooling) | 2026-11-12 |
| D63 | MerchantUser emails are unique across all merchants; one person working at several merchants → future (chains) | 2026-11-12 |
| D64 | Client-facing API: REST/JSON over HTTPS for all reads and commands; Server-Sent Events for live calendar and booking-status updates; no webhooks/gRPC/GraphQL in the MVP (ADR-001) | 2026-11-12 |
| D65 | One versioned API (`/v1`) with zones: public (`/merchants`, GET only), booking flow (`/bookings`), customer (`/me`), merchant management (`/manage/merchants/{merchant_id}`) (ADR-002) | 2026-11-12 |
| D66 | Auth: bearer tokens — short-lived JWT access + rotating refresh tokens for Customers and MerchantUsers; guests get a booking token for their hold and single-purpose link tokens/SMS codes after checkout (ADR-003) | 2026-11-12 |
| D67 | API conventions: transitions as POST action sub-resources; Idempotency-Key on every POST (24 h); optimistic concurrency via version/ETag; RFC 9457 errors with stable codes; cursor pagination; cross-tenant access returns 404 | 2026-11-12 |
| D68 | Events are published reliably via a transactional outbox: outbox row written in the same DB transaction as the change, relay publishes to the broker; at-least-once delivery, consumers idempotent by event id and aggregate version (ADR-004) | 2026-11-12 |
| D69 | SSE fan-out via per-recipient pub/sub topics (`customer:{id}`, `merchant:{id}`); SSE servers subscribe for their connected users, filter by role, replay from a shared store on reconnect; no sticky sessions (ADR-005) | 2026-11-12 |
| D70 | Architecture: microservices behind one API gateway — Identity, Merchant Catalog, Booking & Payments, Availability, Discovery/Search, Reviews, Notifications, Scheduler, Realtime/SSE gateway; Booking and Wallet/Ledger form one **Booking & Payments** service so each transition is one local ACID transaction (ADR-006) | 2026-11-15 |
| D71 | Data stores: PostgreSQL database per service (sync standby in another AZ), Redis for rate limits, slot cache and SSE pub/sub/replay; marketplace search on PostgreSQL (PostGIS + full-text) (ADR-007) | 2026-11-15 |
| D72 | Messaging: Kafka is the durable event stream for outbox events; Redis Pub/Sub + Streams carry SSE fan-out and replay — resolves ADR-004 Q1 and ADR-005 Q1 (ADR-008) | 2026-11-15 |
| D73 | Merchant Catalog owns configuration and publishes versioned snapshots; Booking & Payments and Availability keep local replicas, no sync catalog call on the booking path (ADR-009) | 2026-11-15 |
| D74 | Timed transitions: effective state evaluated in Booking & Payments on every read/write; a Scheduler service keeps durable timers and calls idempotent internal commands; also drives reminders (ADR-010) | 2026-11-15 |
| D75 | Deployment: cloud-agnostic, containers on Kubernetes in one EU region across 3 AZs; managed PostgreSQL/Kafka/Redis; backups + DR copy in a second EU region | 2026-11-15 |
| D76 | Notification channels refined: "in-app" = booking lists and status inside the apps, kept live via SSE; no in-app notification inbox in the MVP (future). Push needs device tokens registered by customer and merchant apps, owned by the Notifications service | 2026-11-15 |
| D77 | When Staff create a booking for a registered Customer, they find the account only by **exact** email or phone match (rate-limited, returns id + name, no browsing); otherwise the booking is a walk-in | 2026-11-15 |
| D78 | The Staff ↔ staff-resource link is stored on the staff **Resource** in Merchant Catalog (`linked_user_id`), not on the MerchantUser; Catalog validates it against its copy of merchant users | 2026-11-15 |
| D79 | Reminder offsets are snapshotted into the booking's terms; changing them affects new bookings only. Re-applying changed offsets to existing bookings' future reminders → future | 2026-11-15 |
| D80 | Identity provider: self-hosted **Keycloak** (realms `customers` and `merchants`) handles registration, login, MFA, sessions and token issuing via OIDC authorization code + PKCE; our Identity service becomes a thin layer for merchant membership, invitations, sign-up, profile/deletion, customer lookup and `identity.events` (ADR-011) | 2026-11-16 |
| D81 | Booking & Payments launches on a single PostgreSQL primary, shard-ready (merchant id everywhere, cell id bits in ids, cell router); scale-out path = merchant cells + customer wallet cells with two-step checkout authorization, activated by measured triggers — resolves architecture Q4 (ADR-012) | 2026-11-18 |
| D82 | "Any available" assignment picks the free resource that is least busy (booked minutes) on the merchant-local day; ties by merchant order. Tables: smallest fitting capacity first | 2026-11-18 |
| D83 | Rescheduling an "any available" booking: the customer chooses to keep the current resource if free (else normal pick, shown before confirming) or a normal any-of pick; a specific choice stays specific | 2026-11-18 |
| D84 | Staff-created bookings and moves must lie within bookable hours unless staff set an explicit override (logged); min_notice doesn't apply; max_horizon applies; allowed while the merchant is live or paused | 2026-11-18 |
| D85 | For approval services, a guest's verify_by is capped at start − approval_margin, so verification always ends before the approval window closes | 2026-11-18 |
| D86 | Allocation model: live allocations only, GiST exclusion constraint (resource, range, other booking), allocation expires_at mirrors the booking's effective deadline, expired overlaps reaped on conflict, released allocations moved to a history table (ADR-013) | 2026-11-18 |
| D87 | Booking transactions: READ COMMITTED + booking row lock + version CAS, global lock order, transaction time as booking clock, materialise lagging effective state on touch, idempotency record as lock (ADR-013) | 2026-11-18 |
| D88 | Booking terms are snapshotted at hold time and charged unchanged at checkout | 2026-11-18 |
| D89 | Partitioning: bookings and booking events monthly by UUIDv7 id; allocation history monthly by range start; idempotency records purged after 24 h | 2026-11-18 |

## 10. Open questions

None at the product-vision level. Requirement-level questions live in [functional requirements §14](../02-requirements/functional-requirements.md#14-open-questions).
