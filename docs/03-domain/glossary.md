# Glossary

> Status: v1.0 — ubiquitous language for docs, APIs and code. Supersedes the preliminary glossary in [Product Vision §7](../01-product/vision-and-purpose.md#7-core-concepts-preliminary-glossary). Entities: [domain model](domain-model.md) · states: [booking lifecycle](booking-lifecycle.md) · money: [ledger model](ledger-model.md).

| Term | Meaning |
|---|---|
| Allocation (ResourceAllocation) | A booking's claim on one resource for one time segment, including buffers. The unit on which no double-booking is enforced. |
| Approval deadline | min(requested_at + approval window, start − approval margin); a PendingApproval booking is auto-rejected when it passes (D59). |
| Approval margin | Per-service minimum time before start by which approval must be decided (D59). |
| Approval window | Per-service time the merchant has to decide after a request (D59). |
| Assigned staff | The MerchantUser linked to a staff resource allocated to the booking (D33); the link is stored on the resource (D78). |
| Available (account / balance) | Spendable (customer) or withdrawable (merchant) funds. |
| Booking | One customer's claim on one service at one time; aggregate root of the booking flow, starting in Held (D48). |
| Booking page / widget | A merchant's own entry point: shareable URL and embeddable snippet (FR-105). |
| BookingEvent | Append-only record of a booking state change or reschedule/move, with time, actor and reason. |
| Booking terms | Immutable snapshot of price, payment terms and policies a booking was made under (FR-311). |
| Buffer | Time before/after a service during which its resources stay allocated. |
| Category | Business type from the fixed marketplace list (D17). |
| Client history | A merchant's view of past bookings for one client: by account, or by normalised email/phone for guests and walk-ins (D53). |
| Contact | Name, email and/or phone embedded on a guest or walk-in booking (D52). |
| CustomerAccount | Global registered customer identity with one wallet (D3, D51). |
| Decision window | See approval window. |
| Deposit | Upfront payment of part of the price (D56). |
| Effective state | A booking's state evaluated against server time: deadlines that have passed count as applied even before the job runs (NFR-107). |
| Expired | Terminal state of a Held booking that ran out or was abandoned (D49). |
| Free-cancellation window | Time before start up to which a customer cancellation is fully refunded (D13). |
| Guest | Unregistered customer; can book only services with no payment terms (FR-701). |
| GuestToken | Single-purpose verification or cancellation link/code for a guest booking (NFR-704). |
| Held | First booking state: resources are held for 10 minutes during checkout (D23, D48). |
| Kept percent | Share of the upfront payment the merchant keeps on a late cancellation (FR-309). |
| Late-booking cutoff | Shorter cutoff used for guest bookings made inside the verification cutoff (D31). |
| Late cancellation | Customer cancellation after the free-cancellation window (FR-509). |
| Ledger account | One of a wallet's sub-accounts (Available, Reserved, Pending) or a system account (D54). |
| Ledger transaction / entry | An immutable, zero-sum money movement and its per-account lines (NFR-105). |
| Merchant (tenant) | A business on the platform; the data-isolation boundary. |
| MerchantPolicy | Versioned merchant-wide settings: guest rules, cutoffs, cancellation policy, limits, reminders. |
| MerchantUser | Owner or Staff login belonging to exactly one merchant (D51). |
| No-show fee | Amount reserved at checkout and captured when the customer doesn't attend (D14, D58). |
| Payment terms | A service's upfront payment, no-show fee and/or wallet-settled remainder. |
| Pending (account) | Merchant sub-account holding upfront payments that may still be refunded (D55). |
| PendingApproval | Booking awaiting the merchant's decision; payment taken, slot held (D11). |
| Prepayment | Upfront payment of the full price (D56). |
| Provisional | Guest booking awaiting verification until verify_by (FR-702). |
| Remainder | Price − upfront payment; settled at venue or via wallet per service (D57). |
| Reschedule | Customer moving a Confirmed or PendingApproval booking to another slot within the free window; same booking, allocations swapped (D50, D61). |
| Reserved (account) | Customer sub-account holding frozen funds for no-show fees and remainders (D54). |
| Resource | Anything with capacity in time: staff, table, room, bay, equipment. |
| Resource requirement | A service's need for one resource: specific, or any of a set (FR-303). |
| Schedule exception | Closure, holiday, time-off or extra hours for a merchant or resource (FR-402). |
| Service | What a customer books: duration, buffers, required resources, price and payment terms. |
| Slot | A computed bookable start time for a service; not stored (FR-403). |
| Tentative allocation set | The new slot's allocations held during a reschedule (D50). |
| Upfront payment | Deposit or prepayment taken at checkout (D56). |
| Verification cutoff | Time before start by which a provisional guest booking must be verified (D4). |
| Virtual funding | System account that is the counterpart of all top-ups and withdrawals (D54). |
| Walk-in client | A client booked by Owner/Staff without an account: name + email or phone (D53). |
| Wallet | A customer's or merchant's set of ledger accounts in the virtual currency (FR-901). |
