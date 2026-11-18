# Functional Requirements

> Status: v1.2 — all open questions resolved (D19–D38). v1.2 (2026-11-15): FR-202, FR-312, FR-602, FR-1001 clarified (D76–D79). v1.1 (2026-11-12): FR-306, FR-307, FR-504, FR-509, FR-510, FR-602, FR-605, FR-607, FR-608, FR-701, FR-1002 updated and FR-313 added for domain decisions D53–D61. Scope comes from [Product Vision & Purpose](../01-product/vision-and-purpose.md); decisions referenced as Dxx live in its decisions log (§9).
> Requirement IDs are stable: never renumber, mark removed ones as *withdrawn*.

## 0. Conventions

- **MUST** = MVP requirement. **SHOULD** = MVP if cheap, otherwise first follow-up.
- Actors: **Owner** (merchant owner), **Staff** (merchant staff member), **Customer** (registered), **Guest**, **System** (scheduled/automatic behaviour).
- All booking times are stored and evaluated in the **merchant's local time zone** and shown to customers in that zone.

## 1. Merchant onboarding & setup

| ID | Requirement | Ref |
|---|---|---|
| FR-101 | An Owner MUST be able to self sign-up a merchant: business name, one category from the fixed list, address/location, country, time zone, contact details. | D17 |
| FR-102 | A new merchant MUST be live in the marketplace immediately after completing required setup (≥1 service, ≥1 resource, opening hours). No operator review. | D17 |
| FR-103 | The Owner MUST be able to edit the merchant profile (description, photos, contact, address) and the booking page appearance (logo, colours). | |
| FR-104 | The Owner MUST be able to temporarily hide the merchant from the marketplace and booking page (e.g. "pause bookings"). Existing bookings are unaffected. | |
| FR-105 | The System MUST provide each merchant a shareable booking-page URL and an embeddable widget snippet. | D1 |

## 2. Merchant users & roles

| ID | Requirement | Ref |
|---|---|---|
| FR-201 | Two roles exist: **Owner** and **Staff**. A merchant has exactly one Owner. | D28 |
| FR-202 | The Owner MUST be able to invite Staff by email, deactivate them, and link each Staff account to a staff resource (the link is kept on the resource). | D28, D78 |
| FR-203 | The Owner can do everything: configuration, policies, all bookings, merchant wallet, reports. | D28 |
| FR-204 | Staff MUST be able to view their own schedule and the bookings assigned to them, mark attendance/no-show, and create bookings (§6). Staff MUST NOT change configuration, policies or the wallet. | D28 |
| FR-205 | Staff MUST be able to manage their own time-off/exceptions; the Owner can manage anyone's. | |

## 3. Resources, services & policies

| ID | Requirement | Ref |
|---|---|---|
| FR-301 | The Owner MUST be able to define **resources** of type staff, table, room, bay or equipment, each with a name and (for tables) seating capacity. | |
| FR-302 | The Owner MUST be able to define **services**: name, description, price, duration, optional buffer before/after, and the **required resources**. | |
| FR-303 | A service MAY require **several resources at the same time** (e.g. practitioner + room). For each requirement the Owner specifies either a specific resource or "any one of" a set. | D19 |
| FR-304 | A service's duration MAY span **multiple days** (e.g. car kept in a bay). Duration counts **opening hours only**: the resource is reserved during open hours on each day and the end time is computed across days. | D25, D30 |
| FR-305 | Restaurant-style services MUST be bookable by **party size**; the system assigns a **single table** whose capacity fits. Table combining is not supported. | D24 |
| FR-306 | For each service the Owner MUST be able to set: at most one **upfront payment** (deposit or prepayment) and a **no-show fee**, each as a **fixed amount or % of price** (or none). | D2, D29, D56 |
| FR-307 | For each service the Owner MUST be able to toggle **manual approval** and set the **decision window** and **approval margin**; the deadline is min(requested_at + window, start − margin). | D11, D59 |
| FR-308 | The Owner MUST be able to set merchant-wide: guest bookings allowed (on/off), **guest verification** (on/off), **verification cutoff**, and a shorter **late-booking cutoff** (§7). | D4, D31 |
| FR-309 | The Owner MUST be able to set one **cancellation policy**: free-cancellation window and the % of deposit/prepayment kept after it. | D13 |
| FR-310 | The Owner SHOULD be able to set booking limits: minimum notice (how soon before start) and maximum horizon (how far ahead). | |
| FR-311 | Changing a service, price or policy MUST NOT change existing bookings: each booking keeps a snapshot of the terms it was made under. | |
| FR-312 | The Owner MUST be able to configure reminder offsets (e.g. 24h and 2h before start). Offsets are part of the booking's terms snapshot: changes apply to new bookings only. | D35, D79 |
| FR-313 | For each service the Owner MUST be able to set **remainder settlement**: at venue or via wallet. Via wallet, the remainder (price − upfront) is reserved at checkout and captured on completion. | D57 |

## 4. Availability

| ID | Requirement | Ref |
|---|---|---|
| FR-401 | The Owner MUST be able to set weekly opening hours for the merchant and working hours per resource. | |
| FR-402 | The Owner/Staff MUST be able to add exceptions: closures, holidays, time-off, extra hours. | |
| FR-403 | The System MUST compute bookable **slots** for a service and date from: working hours, exceptions, existing bookings and holds, buffers, required resources, booking limits. | |
| FR-404 | A customer MUST be able to choose a specific staff member or "any available" where the service allows it. | |
| FR-405 | Shown availability MUST reflect current bookings and holds (no stale slots that are already taken, beyond a short propagation delay — see NFRs). | |
| FR-406 | Slot computation for multi-day services MUST skip closed hours and reserve the resource only during opening hours on each day spanned. | D25, D30 |

## 5. Booking lifecycle (customer)

| ID | Requirement | Ref |
|---|---|---|
| FR-501 | A Customer or Guest MUST be able to book one service per booking, from the marketplace or a merchant booking page/widget. | D1, D19 |
| FR-502 | Selecting a slot MUST create a **hold** on all required resources, valid for **10 minutes** (platform-wide). An expired hold is released automatically. | D23 |
| FR-503 | The System MUST guarantee a resource is never double-booked: two holds/bookings cannot overlap on the same resource, including buffers. | Vision §6 (2) |
| FR-504 | On checkout the System MUST, in one step: take the upfront payment, reserve the no-show fee and any wallet-settled remainder, and create the booking. If the available balance doesn't cover it, the booking MUST fail and the hold remain until expiry. | D2, D14 |
| FR-505 | A booking by a registered Customer with no manual approval MUST be **confirmed instantly**. | D12 |
| FR-506 | A booking for a service with manual approval MUST enter **Pending approval**: slot held, payment taken. | D11 |
| FR-507 | Booking states: Held → (Provisional \| Pending approval \| Confirmed) → Completed \| No-show \| Cancelled \| Rejected. Every transition MUST be recorded with time and actor. | |
| FR-508 | A Customer MUST be able to view upcoming and past bookings across all merchants. | D3 |
| FR-509 | A Customer MUST be able to **cancel**. Within the free-cancellation window: full refund and reservation released. After it: the policy % of the upfront payment is kept by the merchant, the rest refunded, reservations released. Cancelling a Pending approval booking is always fully refunded. | D13, D14, D60 |
| FR-510 | A Customer MUST be able to **reschedule** a Confirmed or Pending approval booking (same service and merchant) within the free-cancellation window; payments and reservations carry over. A pending booking stays pending with a new approval deadline. After the window, rescheduling is not allowed. | D20, D61 |
| FR-511 | Rescheduling MUST hold the new slot before releasing the old one, so the customer never loses both. | D20 |

## 6. Merchant-side booking management

| ID | Requirement | Ref |
|---|---|---|
| FR-601 | Owner/Staff MUST see a calendar of bookings per day/week, per resource. | |
| FR-602 | Owner/Staff MUST be able to create a booking for a registered Customer or for a walk-in/phone client (name + at least one of email/phone). Staff-created bookings take **no wallet payments or reservations**, settle at the venue, and are confirmed immediately. A registered Customer is found only by exact email or phone match; otherwise the booking is a walk-in. | D21, D53, D77 |
| FR-603 | The Owner, or the Staff member the booking is assigned to, MUST be able to **approve or reject** pending bookings. Reject → full refund, reservation released, slot freed. | D11, D33 |
| FR-604 | Owner/Staff MUST be able to cancel a booking; a merchant-initiated cancellation MUST fully refund the customer and release reservations. | |
| FR-605 | Owner/Staff MUST be able to mark a booking **Completed** or **No-show**. No-show → upfront payment kept, no-show fee captured, remainder reservation released. Completed → remainder captured, no-show reservation released. | D14, D57, D58 |
| FR-606 | Owner/Staff MUST be able to move or extend a booking (e.g. repair job overrun) if resources are free. | |
| FR-607 | Owner/Staff MUST be able to see a client's history with their merchant (past bookings, no-shows): by account for registered Customers; for guests and walk-ins, grouped by normalised email or phone. | D53 |
| FR-608 | The System MUST auto-complete a booking not marked Completed/No-show **24h after its end**, with the same money effects as FR-605 Completed. | D32, D57 |

## 7. Guest bookings & verification

| ID | Requirement | Ref |
|---|---|---|
| FR-701 | A Guest MUST be able to book only services with **no** payment terms (no upfront payment, no-show fee or wallet-settled remainder), giving name, email and phone. | D3, D4, D57 |
| FR-702 | If the merchant has verification on, a guest booking MUST start as **Provisional** and a verification link (email) and code (SMS) MUST be sent, stating the cutoff time. | D4, D22 |
| FR-703 | Verifying MUST move the booking to Confirmed (or Pending approval if the service requires it). | D4, D11 |
| FR-704 | The System MUST auto-cancel a Provisional booking still unverified at the cutoff and release the slot. | D4 |
| FR-705 | **Late guest booking**: if a guest books a slot that starts sooner than the verification cutoff, the merchant's shorter **late-booking cutoff** applies instead. The guest MUST be told immediately that the booking expires unless verified before that cutoff. | D31 |
| FR-707 | Owner/Staff MUST be able to verify any Provisional guest booking themselves (e.g. after calling the guest). | D31, D37 |
| FR-708 | When verification is on, slots starting sooner than the late-booking cutoff MUST NOT be bookable by guests; they are offered to register or contact the merchant. | D38 |
| FR-706 | A Guest MUST be able to cancel via a secure link in their confirmation email. Guests cannot reschedule; they cancel and rebook. | D34 |

## 8. Approval deadline

| ID | Requirement | Ref |
|---|---|---|
| FR-801 | The System MUST auto-reject a Pending approval booking at its approval deadline, fully refund it and release the slot. | D11 |
| FR-802 | Pending approvals MUST be notified to, and actionable by, the Owner and the assigned Staff member. | D28, D33 |

## 9. Wallets & payments (virtual)

| ID | Requirement | Ref |
|---|---|---|
| FR-901 | Every registered Customer and every merchant MUST have one wallet in the single virtual currency. | D7 |
| FR-902 | Customers MUST be able to deposit (top up) and withdraw virtual funds; the Owner MUST be able to withdraw from the merchant wallet. | D7 |
| FR-903 | A wallet MUST show **balance**, **reserved** and **available** (= balance − reserved), plus a transaction history. | D14 |
| FR-904 | Withdrawals MUST NOT exceed the available balance. | D14 |
| FR-905 | Each money movement (top-up, withdrawal, payment, refund, reservation, release, fee capture) MUST be a recorded, immutable transaction linked to its booking where relevant. | Vision §6 (7) |
| FR-906 | Booking payments go customer → merchant directly (no platform fee in the MVP), but the transaction model MUST allow a fee line later. | D15 |
| FR-907 | The money boundary MUST be behind an interface so a real payment provider can replace it. | D7 |

## 10. Notifications

| ID | Requirement | Ref |
|---|---|---|
| FR-1001 | Channels: **email** and **in-app/push**. **SMS** is used only for guest verification codes. "In-app" means booking lists and status in the apps, updated live; there is no notification inbox in the MVP. Apps register device tokens for push. | D22, D76 |
| FR-1002 | Customer/Guest notifications: booking confirmed, provisional (verify now), pending approval, approved, rejected/auto-rejected, cancelled (by whom, refund amount), rescheduled, reminder before start, no-show fee charged, remainder charged. | D57 |
| FR-1003 | Merchant notifications: new booking, booking pending approval (with deadline), cancellation, reschedule, auto-cancelled/auto-rejected booking. | |
| FR-1004 | Reminders MUST be sent at the merchant-configured offsets (FR-312). | D35 |
| FR-1005 | Users SHOULD be able to opt out of non-essential notifications (reminders) but not transactional ones. | |

## 11. Marketplace & discovery

| ID | Requirement | Ref |
|---|---|---|
| FR-1101 | Customers and Guests MUST be able to browse merchants by category and search by name/keyword and location (near me / city). | D1 |
| FR-1102 | A merchant page MUST show profile, services with price/duration/payment terms, rating, reviews and available slots. | |
| FR-1103 | Search by availability ("free at date/time") is **future**. | D27 |

## 12. Reviews & ratings

| ID | Requirement | Ref |
|---|---|---|
| FR-1201 | A registered Customer MUST be able to leave one review per **Completed** booking: 1–5 stars and optional text. | D26 |
| FR-1202 | The Owner MUST be able to post one public reply per review. | D26 |
| FR-1203 | A merchant's rating is the average of its reviews and MUST be shown with the review count. | D26 |
| FR-1204 | No moderation tools in the MVP (operator tooling is future). | D16, D26 |

## 13. Customer account

| ID | Requirement | Ref |
|---|---|---|
| FR-1301 | A person MUST be able to register with email (+ password) and manage profile (name, phone). One account works across all merchants. | D3 |
| FR-1302 | A Customer MUST be able to manage notification preferences (FR-1005). | |
| FR-1303 | Guest bookings are NOT attached to an account later registered with the same email. | D36 |

## 14. Open questions

| # | Question | Status | Raised |
|---|---|---|---|
| Q1 | Multi-day services during closed hours | Resolved → D30 | 2026-11-11 |
| Q2 | Guest booking after the verification cutoff has passed | Resolved → D31 | 2026-11-11 |
| Q3 | Can guests reschedule? | Resolved → D34 | 2026-11-11 |
| Q4 | Bookings never marked Completed/No-show | Resolved → D32 | 2026-11-11 |
| Q5 | Who approves pending bookings | Resolved → D33 | 2026-11-11 |
| Q6 | Reminder timing | Resolved → D35 | 2026-11-11 |
| Q7 | Attach guest bookings to a later account | Resolved → D36 | 2026-11-11 |
| Q8 | Manual verification scope | Resolved → D37 | 2026-11-11 |
| Q9 | Guest booking inside the late-booking cutoff | Resolved → D38 | 2026-11-11 |
