# ADR-002: API surface — one versioned API with scoped zones

- **Status:** Accepted
- **Date:** 2026-11-12
- **Decision:** D65

## Context

Three kinds of caller (anonymous/guest, Customer, Owner/Staff) use overlapping data: e.g. services are shown publicly and edited by the Owner. Tenant isolation must be enforced by default (NFR-601), and the design should stay small.

## Options

| Option | Pros | Cons |
|---|---|---|
| **One API, zones by path prefix** | One contract, one gateway; tenant id visible in merchant paths; easy to check per zone | Some resources appear in two zones (public vs management view) |
| Separate APIs / BFF per client | Clear ownership per app | More contracts and duplication before we know we need them |
| Same path, view depends on caller | Fewer paths | Responses vary by token; easy to leak data; hard to cache |

## Decision

One API under `/v1` with zones: `/auth`, public `/categories` + `/merchants` (GET only), booking flow `/bookings`, customer `/me`, and merchant management `/manage/merchants/{merchant_id}`. In the management zone the path `merchant_id` must match the token's merchant; mismatches return 404.

## Consequences

- Zone → identity type is a single, central authorisation rule; tenant checks hang off one path parameter.
- Public and management views of the same entity are separate endpoints with separate representations.
- `merchant_id` in the path keeps the door open for users working at several merchants (chains, future).
- Per-client BFFs can still be added in front later if apps need tailored payloads.
