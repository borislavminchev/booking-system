# API Conventions

> Status: v0.3 — first, deliberately minimal pass (D64–D67); v0.3: authentication via Keycloak OIDC (D80). Grow it only when a requirement needs more.
> Companion: [Endpoint catalogue](endpoint-catalogue.md) · ADRs [001](../07-decisions/ADR-001-api-style.md), [002](../07-decisions/ADR-002-api-surface.md), [003](../07-decisions/ADR-003-api-authentication.md).

## 1. Styles

| Interaction | Style | Why |
|---|---|---|
| All reads and commands (all clients) | **REST over HTTPS, JSON** | One style for every client; cacheable public reads; simple tooling (D64) |
| Live updates: merchant calendar, booking status in apps | **Server-Sent Events (SSE)** | One-way server → client push meets NFR-503 (≤2 s) without a bidirectional protocol (D64) |
| Webhooks, gRPC, GraphQL | Not in MVP | No external integrations; internal APIs are decided in [05-architecture](../05-architecture/) |

## 2. Surface and URL layout (D65)

One API, one base URL, major version in the path: `https://api.<platform-domain>/v1`.

| Zone | Prefix | Caller | Tenant scope |
|---|---|---|---|
| Auth | `/v1/auth/...` (sign-up, invitations); login/tokens at the identity provider (§8) | Anyone | — |
| Public | `/v1/categories`, `/v1/merchants/...` (GET only) | Anyone, incl. widget | Public merchant data only |
| Booking flow | `/v1/bookings/...` | Customer, Guest (booking token) | The booking's own data |
| Customer | `/v1/me/...` | Customer | The caller's own account (NFR-602) |
| Merchant management | `/v1/manage/merchants/{merchant_id}/...` | Owner / Staff | `merchant_id` in the path MUST equal the token's merchant (NFR-601) |

- The public and management zones are kept apart so a URL never returns different views depending on who calls it: public `GET /v1/merchants/{id}/services` lists active services; `GET /v1/manage/merchants/{id}/services` lists all, incl. inactive and versions.
- Resource names are plural nouns; path segments and JSON fields are `snake_case` (same as the domain docs).
- **State transitions are action sub-resources:** `POST /v1/bookings/{id}/cancel`, `/checkout`, `/approve`, … — one endpoint per transition in the [booking lifecycle](../03-domain/booking-lifecycle.md#3-transitions). No generic `PATCH state`.
- The widget and booking page are served from the platform's own domain (iframe), so they call the API same-origin; third-party sites never get API access (NFR-706).

## 3. Data formats

| Kind | Format | Example |
|---|---|---|
| Ids | Opaque strings (UUIDv7) | `"0193a1c2-…"` |
| Instants | RFC 3339 in UTC | `"2026-11-20T08:30:00Z"` |
| Merchant-local values | `time_zone` (IANA) on the merchant; local dates/times as ISO strings | `"date": "2026-11-20"`, `"opens": "09:00"` |
| Durations | Integer minutes, field suffix `_min` | `"duration_min": 45` |
| Money | Object: integer minor units + currency | `{"amount": 2500, "currency": "VCR"}` |
| Enums | lower `snake_case` | `"state": "pending_approval"` |

- Booking states: `held`, `provisional`, `pending_approval`, `confirmed`, `completed`, `no_show`, `cancelled`, `rejected`, `expired`.
- Every read returns the **effective state** (NFR-107): a Held booking past `hold_expires_at` is returned as `expired` even before the job has run.
- Slot queries take a **local date** (`?date=2026-11-20`) interpreted in the merchant's time zone; returned slots carry UTC `start`/`end`.

## 4. Requests and responses

- **Single resource:** the object itself. **Collections:** `{"data": [...], "next_cursor": "…" | null}`.
- **Pagination:** cursor-based, `?limit=` (default 25, max 100) and `?cursor=`.
- **Filtering:** plain query parameters (`?state=confirmed&from=…&to=…`). No generic query language.
- **Partial updates:** `PATCH` with a JSON merge-patch body. Editing a versioned object (service, policy) creates a new version and returns it (FR-311).
- **Creation:** `201 Created` + `Location` header + the resource.

## 5. Idempotency (NFR-106)

- Every `POST` (commands and creations) MUST carry `Idempotency-Key: <client-generated UUID>`. Missing → `400 idempotency_key_required`.
- The server stores key → response for **24 h**, scoped to the caller. A retry with the same key and body returns the original response (same status, same body). Same key with a different body → `422 idempotency_key_reused`.
- `PUT`, `PATCH` and `DELETE` are idempotent by definition and don't need the header.

## 6. Concurrency (NFR-109)

- Bookings and versioned configuration carry `version`, also sent as `ETag`.
- `PATCH`/`PUT` on configuration SHOULD send `If-Match: <version>`; a mismatch → `412 version_mismatch`.
- Booking action endpoints MAY send `If-Match`. With or without it, the server applies the transition by compare-and-set; if another transition won, the loser gets `409 invalid_state` with the booking's current state in the error body.

## 7. Errors

Errors use **Problem Details** (RFC 9457), `Content-Type: application/problem+json`, plus a stable machine-readable `code`:

```json
{
  "type": "https://docs.<platform-domain>/errors/slot_taken",
  "title": "This slot was just taken",
  "status": 409,
  "code": "slot_taken",
  "detail": "Another booking claimed the slot while you were choosing.",
  "alternatives": [{"start": "2026-11-20T09:00:00Z", "end": "2026-11-20T09:45:00Z"}]
}
```

| Status | When | Common `code`s |
|---|---|---|
| 400 | Malformed request | `validation_failed`, `idempotency_key_required` |
| 401 | Missing/invalid/expired token | `unauthenticated`, `token_expired` |
| 403 | Authenticated but not allowed (role, other tenant) | `forbidden` |
| 404 | Not found **or** not visible to the caller (never reveal other tenants' ids) | `not_found` |
| 409 | Conflict with current state | `slot_taken`, `invalid_state`, `hold_expired`, `reschedule_not_allowed` |
| 412 | `If-Match` mismatch | `version_mismatch` |
| 422 | Valid but breaks a business rule | `insufficient_funds`, `outside_booking_window`, `guest_not_allowed`, `idempotency_key_reused`, `merchant_not_ready` |
| 429 | Rate limit / abuse limit (NFR-604, NFR-707) | `rate_limited`, `too_many_holds`, `challenge_required` |
| 5xx | Server fault; safe to retry with the same Idempotency-Key | — |

Cross-tenant access returns **404**, not 403, so ids of other merchants' data can't be probed (NFR-601, NFR-603).

## 8. Authentication (D66, D80)

Customer and MerchantUser login, MFA, sessions and token issuing are handled by **Keycloak** ([ADR-011](../07-decisions/ADR-011-identity-provider.md)) at `https://auth.<platform-domain>/realms/{customers|merchants}`, using standard OpenID Connect: **authorization code + PKCE** for every client (web, mobile via the system browser, widget via popup). No password grant.

| Token | Who | How obtained | Lifetime | Scope |
|---|---|---|---|---|
| Access token (JWT) | Customer or MerchantUser | Keycloak token endpoint (code exchange / refresh) | 15 min | Customer: `/v1/me`, `/v1/bookings`; MerchantUser: its merchant + role (`owner`/`staff`) |
| Refresh token (rotating, single use) | Same | Keycloak token endpoint | 30-day max session, rotated on each use, reuse revokes the session, revoked on logout | Only the Keycloak token endpoint |
| Booking token (opaque) | Guest during checkout | Returned when a guest creates a hold | Until the hold ends | That one Held booking: read, checkout, abandon |
| Guest link token / SMS code | Guest after checkout | Sent by email/SMS | Until verify_by (verify) or booking end (cancel) | One purpose on one booking (NFR-704) |

- All tokens are sent as `Authorization: Bearer <token>`, except guest verify/cancel tokens and codes, which go in the request body of the matching action.
- Customer and MerchantUser are separate identities (D51) in separate realms; the token issuer tells which one it is. Merchant tokens carry `merchant_id` and `role` claims.
- Owner login requires MFA (NFR-701), enforced by the merchants realm's login flow (OTP or WebAuthn).
- Wallet withdrawals require a recent login (≤5 min, `auth_time` claim); otherwise `401 reauthentication_required` and the client re-authenticates with `max_age=300` (NFR-708).
- Authorisation is checked server-side on every request (NFR-703): zone → identity type, `merchant_id` → tenant, role → action (see the permissions column in the catalogue).

## 9. Real-time updates (SSE)

- Endpoints: `GET /v1/me/events` (customer) and `GET /v1/manage/merchants/{merchant_id}/events` (Owner/Staff; Staff receive only events for bookings assigned to them or created by them).
- Clients use a fetch-based SSE client so the `Authorization` header can be sent (browser `EventSource` can't set headers; a stream-ticket fallback can be added later if needed).
- Events are **thin notifications**; clients refetch the resource:

```text
id: 01J9Z3…
event: booking.updated
data: {"booking_id":"0193…","state":"confirmed","version":4}
```

- Event types (initial): `booking.created`, `booking.updated` (any state change, reschedule, move), `wallet.updated`.
- Resume with `Last-Event-ID`; the server keeps a short replay buffer (≥5 min). A heartbeat comment is sent every 20 s. If the buffer is exceeded the client refetches the list.
- SSE is best-effort; REST reads remain the source of truth.
- Server side: events come from the transactional outbox ([ADR-004](../07-decisions/ADR-004-transactional-outbox.md)) and reach the right SSE server via per-user pub/sub topics ([ADR-005](../07-decisions/ADR-005-sse-fan-out.md)); replay on reconnect uses a shared store, so any server can resume a stream.

## 10. Rate limits and other headers

- Limits per caller, per tenant and per IP (NFR-604, NFR-707). Responses carry `RateLimit-Limit`, `RateLimit-Remaining`, `RateLimit-Reset`; `429` adds `Retry-After`.
- Hold creation has an extra per-client/IP cap on concurrent active holds (`429 too_many_holds`); suspected abuse returns `429 challenge_required` with a challenge to solve.
- Every response carries `X-Request-Id` (also accepted from the client) for tracing.
- Public GETs (merchant profile, services, reviews) may be cached (`Cache-Control: max-age=60`); slot responses may be cached for at most 1 s so availability stays within the ≤2 s freshness target (NFR-501); checkout always re-validates (NFR-502).

## 11. Versioning and evolution

- Additive changes (new fields, endpoints, enum values, error codes) are **non-breaking**; clients MUST ignore unknown fields and handle unknown enum values gracefully.
- Breaking changes go to `/v2`; `/v1` stays available for a deprecation period announced via `Deprecation`/`Sunset` headers.
- This document starts minimal on purpose: new styles (webhooks, a public partner API, GraphQL for the apps) or features (bulk endpoints, field selection) are added through new ADRs when a requirement needs them.

## 12. Open questions

None.

## 13. Change log

| Version | Date | Change |
|---|---|---|
| v0.1 | 2026-11-12 | First minimal version (D64–D67) |
| v0.2 | 2026-11-12 | SSE server-side flow linked to ADR-004/005 (D68, D69) |
| v0.3 | 2026-11-16 | Authentication through Keycloak OIDC (D80, ADR-011) |
