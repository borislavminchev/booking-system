# Endpoint Catalogue

> Status: v0.4 — v0.4 adds `keep_resource` and `override_hours` (D83, D84); first, minimal set covering the MVP FRs; v0.2 adds device registration, customer lookup and moves the staff link (D76–D78); v0.3 moves login/tokens to Keycloak OIDC (D80). Conventions (errors, idempotency, auth, SSE) are in [API conventions](api-conventions.md). Transitions Tn refer to [booking lifecycle §3](../03-domain/booking-lifecycle.md#3-transitions).
> All paths are relative to `/v1`. Every `POST` needs an `Idempotency-Key` header.

**Who** column: `Any` = no token · `Cust` = Customer · `Guest` = booking token / guest link token · `Owner` · `Staff` (= Owner or Staff; Staff limited as noted).

## 1. Auth

Login, registration, MFA, token refresh and logout use **Keycloak's standard OpenID Connect endpoints** (D80, [ADR-011](../07-decisions/ADR-011-identity-provider.md)), not `/v1`:

| Endpoint (per realm `customers` / `merchants`) | Purpose | Ref |
|---|---|---|
| `GET …/protocol/openid-connect/auth` (+ PKCE; `kc_action=register` / registration page for customers) | Hosted login, customer self-registration, MFA (always for Owners), re-auth with `max_age=300` | FR-1301, NFR-701, NFR-702, NFR-708 |
| `POST …/protocol/openid-connect/token` | Code exchange; refresh with rotation | ADR-003 |
| `GET …/protocol/openid-connect/logout` · `POST …/revoke` | Logout, revoke refresh token | |
| `GET …/protocol/openid-connect/certs` | JWKS for token validation | |

Our own `/v1/auth` endpoints (thin Identity service):

| Method & path | Purpose | Who | Ref |
|---|---|---|---|
| `POST /auth/merchant-signups` | Self sign-up: create Merchant (status `setup`) in Catalog + its Owner in Keycloak; Owner sets password and MFA on first login | Any | FR-101, NFR-701 |
| `POST /auth/invitations/{token}/accept` | Staff accepts invite; activates the account and redirects to Keycloak to set a password | Any (invite token) | FR-202 |

## 2. Public discovery

| Method & path | Purpose | Who | Ref |
|---|---|---|---|
| `GET /categories` | Fixed category list | Any | D17 |
| `GET /merchants?category=&q=&near=lat,lng&radius_km=&city=` | Search live merchants (search index, may lag ≤5 min) | Any | FR-1101, NFR-504 |
| `GET /merchants/{merchant_id}` | Public profile: details, branding, rating + count, time_zone | Any | FR-1102, FR-1203 |
| `GET /merchants/by-slug/{slug}` | Resolve booking-page slug → public profile | Any | FR-105 |
| `GET /merchants/{merchant_id}/services` | Active services with price, duration, payment terms, choosable staff | Any | FR-1102 |
| `GET /merchants/{merchant_id}/services/{service_id}/slots?date=&staff_id=&party_size=&for=guest` | Bookable slots for one local date (read model, ≤2 s fresh). `for=guest` applies guest rules (FR-708) | Any | FR-403–406, FR-708, D59 |
| `GET /merchants/{merchant_id}/reviews` | Reviews with merchant replies | Any | FR-1102, FR-1202 |

Slot item: `{"start", "end", "resource_options": {"<requirement_id>": ["<resource_id>", …]}}` — lets the client offer "specific staff or any available" (FR-404).

## 3. Booking flow (customer & guest)

| Method & path | Purpose | Who | Lifecycle | Ref |
|---|---|---|---|---|
| `POST /bookings` | **Hold a slot**: `merchant_id`, `service_id`, `start`, `party_size?`, `resource_choices?` (`{requirement_id: resource_id}`), `origin`. Returns the booking in `held` with `hold_expires_at`; for guests also a `booking_token` | Cust, Any (guest) | T1, T1a | FR-501, FR-502, NFR-707 |
| `GET /bookings/{booking_id}` | Booking with effective state, terms snapshot, deadlines, allocations | Cust (own), Guest | — | FR-508 |
| `POST /bookings/{booking_id}/checkout` | Complete checkout. Guest body: `contact` (name, email, phone). Result: `confirmed`, `pending_approval` or `provisional` (+ `verify_by`) | Cust, Guest | T3, T4, T5 | FR-504–506, FR-702, FR-705 |
| `POST /bookings/{booking_id}/abandon` | Release the hold now | Cust, Guest | T6 | FR-502, D49 |
| `POST /bookings/{booking_id}/verify` | Guest verification: body `{"token"}` (email link) or `{"code"}` (SMS) | Any (body token) | T7 | FR-703, NFR-704 |
| `POST /bookings/{booking_id}/cancel` | Cancel. Guests send `{"token"}` from the cancel link. Response includes refund/kept amounts | Cust, Any (body token) | T8, T11–T13 | FR-509, FR-706, D60 |
| `POST /bookings/{booking_id}/reschedule` | Hold a new slot as a tentative set (`start`, `resource_choices?`, `keep_resource?` — keep the current resource if free, D83); returns `reschedule.expires_at` and `resources_kept` | Cust | T15 step 1 | FR-510, FR-511, D50, D61 |
| `POST /bookings/{booking_id}/reschedule/confirm` | Swap tentative ↔ current | Cust | T15 step 2 | FR-510 |
| `DELETE /bookings/{booking_id}/reschedule` | Drop the tentative set, keep original slot | Cust | T15 step 3 | D50 |

Booking response (abridged):

```json
{
  "id": "0193…", "merchant_id": "…", "service_id": "…", "service_version": 3,
  "state": "confirmed", "version": 4,
  "start": "2026-11-20T08:30:00Z", "end": "2026-11-20T09:15:00Z", "time_zone": "Europe/Sofia",
  "party_size": null,
  "allocations": [{"resource_id": "…", "resource_name": "Maria", "type": "staff"}],
  "terms": {
    "price": {"amount": 5000, "currency": "VCR"},
    "upfront": {"kind": "deposit", "amount": {"amount": 1000, "currency": "VCR"}},
    "no_show_fee": {"amount": 1500, "currency": "VCR"},
    "remainder": {"settlement": "at_venue", "amount": {"amount": 4000, "currency": "VCR"}},
    "free_cancellation_until": "2026-11-19T08:30:00Z",
    "late_cancellation_kept_percent": 50,
    "requires_approval": false
  },
  "hold_expires_at": null, "verify_by": null, "approval_deadline": null,
  "reschedule": null
}
```

## 4. Customer account (`/me`)

| Method & path | Purpose | Who | Ref |
|---|---|---|---|
| `GET /me` · `PATCH /me` | Profile (name, phone) | Cust | FR-1301 |
| `DELETE /me` | Delete account; personal data erased/pseudonymised | Cust | NFR-804 |
| `GET /me/notification-preferences` · `PUT …` | Reminder opt-out | Cust | FR-1005, FR-1302 |
| `GET /me/bookings?scope=upcoming\|past&cursor=` | Own bookings across merchants | Cust | FR-508 |
| `POST /me/bookings/{booking_id}/review` | One review per Completed booking (stars, text?) | Cust | FR-1201 |
| `GET /me/wallet` | Balance, reserved, available | Cust | FR-903 |
| `GET /me/wallet/transactions?cursor=` | Transaction history (linked booking ids) | Cust | FR-903, FR-905 |
| `POST /me/wallet/deposits` | Virtual top-up (`amount`) | Cust | FR-902 |
| `POST /me/wallet/withdrawals` | Virtual withdrawal ≤ available; recent re-auth required | Cust | FR-902, FR-904, NFR-708 |
| `GET /me/events` | SSE: own booking and wallet updates | Cust | NFR-503 |
| `PUT /me/devices/{device_id}` · `DELETE …` | Register / remove a push device token (platform, token) | Cust | FR-1001, D76 |

## 5. Merchant management (`/manage/merchants/{merchant_id}`)

Prefix omitted below: every path starts with `/manage/merchants/{merchant_id}`.

### 5.1 Merchant, policy, users

| Method & path | Purpose | Who | Ref |
|---|---|---|---|
| `GET ` · `PATCH ` (prefix itself) | Merchant profile, contact, address, branding | Staff (read), Owner | FR-103 |
| `POST /go-live` | `setup` → `live` (checks ≥1 service, ≥1 resource, opening hours; else `422 merchant_not_ready`) | Owner | FR-102 |
| `POST /pause` · `POST /resume` | `live` ⇄ `paused` | Owner | FR-104 |
| `GET /booking-page` | Booking-page URL, slug, widget snippet | Owner | FR-105 |
| `GET /policy` · `PUT /policy` | Current policy; PUT creates a new version | Staff (read), Owner | FR-308–310, FR-312 |
| `GET /users` | Owner + Staff list | Owner | FR-202 |
| `POST /users/invitations` | Invite Staff by email | Owner | FR-202 |
| `POST /users/{user_id}/deactivate` | Deactivate Staff | Owner | FR-202 |

### 5.2 Resources, services, schedules

| Method & path | Purpose | Who | Ref |
|---|---|---|---|
| `GET /resources` · `POST /resources` | List / create (type, name, seating_capacity) | Staff (read), Owner | FR-301 |
| `GET` · `PATCH /resources/{resource_id}` | Read / update (incl. `active`; `linked_user_id` for staff resources) | Staff (read), Owner | FR-301, FR-202, D78 |
| `GET /services` · `POST /services` | List (incl. inactive) / create with `requirements[]`, payment terms, approval settings | Staff (read), Owner | FR-302–307, FR-313 |
| `GET` · `PATCH /services/{service_id}` | Read / update → new version | Staff (read), Owner | FR-311 |
| `GET` · `PUT /opening-hours` | Weekly opening hours (local time) | Staff (read), Owner | FR-401 |
| `GET` · `PUT /resources/{resource_id}/working-hours` | Weekly working hours | Staff (read), Owner | FR-401 |
| `GET /schedule-exceptions?from=&to=&resource_id=` | List closures / time-off / extra hours | Staff | FR-402 |
| `POST /schedule-exceptions` · `DELETE /schedule-exceptions/{id}` | Add / remove (merchant- or resource-level) | Owner; Staff for own resource | FR-205, FR-402 |

### 5.3 Bookings

| Method & path | Purpose | Who | Lifecycle | Ref |
|---|---|---|---|---|
| `GET /bookings?from=&to=&resource_id=&state=` | Calendar per day/week/resource (Staff: default to own) | Staff | — | FR-601, FR-204 |
| `GET /bookings/{booking_id}` | Booking incl. event history | Staff | — | FR-507 |
| `GET /slots?service_id=&date=&staff_id=&party_size=` | Slots for staff-side booking (ignores min notice / guest rules) | Staff | — | FR-602 |
| `POST /bookings` | Create booking for a registered customer (`customer_id`) or walk-in (`contact`); no payments, confirmed; `override_hours?` to book outside bookable hours (D84) | Staff | T2 | FR-602, D21, D53 |
| `POST /bookings/{booking_id}/approve` · `/reject` | Decide on PendingApproval | Owner, assigned Staff | T9, T10 | FR-603, FR-802 |
| `POST /bookings/{booking_id}/verify` | Verify a Provisional guest booking | Staff | T7 | FR-707 |
| `POST /bookings/{booking_id}/cancel` | Merchant cancellation (full refund) | Staff | T8, T11, T14 | FR-604 |
| `POST /bookings/{booking_id}/complete` · `/no-show` | Mark attendance | Staff | T17, T18 | FR-605 |
| `POST /bookings/{booking_id}/move` | Move / extend: new `start`, `end?`, `resource_choices?`, `override_hours?` (D84) | Staff | T16 | FR-606 |
| `GET /clients?q=` | Client list (registered + guest/walk-in groups) | Staff | — | FR-607, D53 |
| `GET /clients/{client_key}/bookings` | One client's history with this merchant | Staff | — | FR-607 |
| `GET /events` | SSE: calendar/booking updates (Staff: own bookings) | Staff | — | NFR-503 |
| `GET /customers/lookup?email=` or `?phone=` | Exact-match lookup of a registered Customer for a staff booking → `{customer_id, name}` or 404; rate-limited, no partial search | Staff | — | FR-602, D77 |
| `PUT /devices/{device_id}` · `DELETE …` | Register / remove the caller's push device token | Staff | — | FR-1001, D76 |

`client_key` is opaque: it encodes either a customer id or a normalised email/phone group, so the merchant never sees other merchants' data (NFR-602).

### 5.4 Wallet & reviews

| Method & path | Purpose | Who | Ref |
|---|---|---|---|
| `GET /wallet` | Available, pending | Owner | FR-903, D55 |
| `GET /wallet/transactions?cursor=` | History | Owner | FR-905 |
| `POST /wallet/withdrawals` | Withdraw ≤ available; recent re-auth | Owner | FR-902, NFR-708 |
| `GET /reviews` | Reviews of this merchant | Staff | FR-1201 |
| `PUT /reviews/{review_id}/reply` | Post/replace the one public reply | Owner | FR-1202 |

## 6. Not in the API

- Scheduled transitions (hold expiry, verification cutoff, approval deadline, auto-complete — T6, T8, T10, T17) run inside the system; clients see them through reads and SSE.
- Notifications (email, push, SMS) are sent by the system; only preferences and push device registration are exposed (FR-1005, D76). No in-app notification inbox in the MVP.
- Platform-operator endpoints: future (D16).

## 7. Open questions

None.

## 8. Change log

| Version | Date | Change |
|---|---|---|
| v0.1 | 2026-11-12 | First minimal catalogue (D64–D67) |
| v0.2 | 2026-11-15 | Push device registration (D76), customer lookup for staff bookings (D77), staff link via resource (D78) |
| v0.3 | 2026-11-16 | Login, registration, MFA, refresh and logout via Keycloak OIDC endpoints (D80) |
| v0.4 | 2026-11-18 | `keep_resource` on reschedule (D83), `override_hours` on staff create/move (D84) |
