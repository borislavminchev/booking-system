# ADR-003: API authentication — bearer tokens, plus scoped guest tokens

- **Status:** Accepted; amended by [ADR-011](ADR-011-identity-provider.md) (2026-11-16): tokens are issued by Keycloak via OIDC authorization code + PKCE
- **Date:** 2026-11-12
- **Decision:** D66

## Context

Callers are web, mobile and an iframe widget embedded on third-party sites. Customers and MerchantUsers are separate identities (D51); Owners need MFA (NFR-701). Guests have no account but must complete checkout for their own hold and later verify or cancel via email/SMS (NFR-704).

## Options

| Option | Pros | Cons |
|---|---|---|
| **Bearer tokens (JWT access + rotating refresh)** | Same mechanism for web, mobile and widget; stateless access checks; no CSRF | Token storage on clients; revocation only at refresh (short access TTL mitigates) |
| Session cookies | Simple for first-party web | Third-party-cookie limits in embedded widgets; CSRF handling; awkward for mobile |

## Decision

- Customers and MerchantUsers: JWT access token (15 min) carrying identity type, id and — for MerchantUsers — merchant id and role; opaque rotating refresh token (30 days), revoked on logout.
- Separate login per identity type (since ADR-011: one Keycloak realm per type, hosted login pages); Owner login always goes through an MFA step.
- Guests: an opaque **booking token** returned with the hold, valid only for that Held booking (read, checkout, abandon); after checkout, single-purpose link tokens / SMS codes sent in the action body for verify and cancel.
- Withdrawals require a login within the last 5 minutes (NFR-708).

## Consequences

- One `Authorization: Bearer` path for all clients; authorisation is identity type + tenant + role.
- Refresh-token rotation and reuse detection are provided by the identity provider (ADR-011).
- If revocation of access tokens becomes necessary, a deny-list or shorter TTL can be added without changing the API.
