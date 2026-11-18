# ADR-011: Identity provider — self-hosted Keycloak

- **Status:** Accepted (amends [ADR-003](ADR-003-api-authentication.md))
- **Date:** 2026-11-16
- **Decision:** D80

## Context

ADR-006 planned an Identity service that implements registration, login, MFA (required for Owners, NFR-701), password hashing, breached-password checks and lockout (NFR-702), JWT issuing with key rotation, rotating refresh tokens with reuse detection, and recent-authentication checks for withdrawals (NFR-708). This is security-critical, generic functionality with no product differentiation, and it carries a lasting maintenance and audit burden. Constraints: two separate identity types with possibly the same email (D51, D63), EU data residency (NFR-802), a cloud-agnostic deployment (D75), and potentially millions of customer accounts.

## Options

| Option | Pros | Cons |
|---|---|---|
| **Self-hosted Keycloak** | Mature open-source OIDC provider: MFA (OTP, WebAuthn), configurable flows, brute-force detection, refresh-token rotation, JWKS key rotation, Organizations; runs in our EU cluster; no per-user fees | Stateful clustered Java service to operate and upgrade; customisation via SPIs/themes; transactional event publishing not built in |
| Managed IdP (Auth0, Zitadel Cloud, Cognito, …) | No IdP operations | Per-MAU pricing at marketplace scale; EU residency and vendor lock-in to check |
| Build our own (previous plan) | Full control, events via our outbox | We own all security-critical auth code, its audits and its bugs |

## Decision

1. **Keycloak** (self-hosted, in the primary EU region, ≥3 nodes across AZs, its own PostgreSQL cluster with sync standby) is the platform's OpenID Connect provider.
2. **Two realms**, matching the two identity types (D51): `customers` and `merchants`. The same email may exist in both; emails are unique within a realm (D63).
3. **Merchant membership**: each merchant user belongs to exactly one merchant (Keycloak Organization or user attribute — detailed in the identity deep-dive). A protocol mapper adds `merchant_id` and `role` (`owner` · `staff`) claims to access tokens.
4. **Flows**: OIDC **authorization code + PKCE** for all clients (web, mobile via system browser, widget via popup). No password grant. Customers self-register on Keycloak's (themed) registration page. Owner logins always require a second factor (conditional flow by role).
5. **Tokens**: access token 15 min; refresh-token rotation with reuse detection (refresh token single use) and 30-day maximum session — as in ADR-003. Gateway and services validate JWTs against the realm's JWKS; the issuer identifies the identity type.
6. **Recent authentication** (NFR-708): clients re-authenticate with `max_age=300`; the wallet API checks the `auth_time` claim.
7. Our **Identity service stays, but thin**: merchant sign-up orchestration (Catalog + Owner account via Keycloak admin API), staff invitations and deactivation, `/me` profile and account deletion, exact-match customer lookup (D77), and publishing `identity.events`.
8. **Events**: a Keycloak event-listener extension forwards user events (register, update, delete, login-relevant changes) to the Identity service, which writes them to its outbox → `identity.events`. Because that hop is not transactional, the Identity service also runs a periodic reconciliation against the Keycloak admin API.
9. **Guest tokens** (booking token, verify/cancel links, SMS codes) are not identities and stay in Booking & Payments (ADR-003).

## Why Keycloak rather than a managed IdP (Auth0 and similar)

Both options remove our own auth code; the choice is between operating the IdP ourselves and paying a vendor to operate it. We chose self-hosted Keycloak because:

| Criterion | Self-hosted Keycloak | Managed IdP | Why it matters for us |
|---|---|---|---|
| **Cost at our scale** | Infrastructure + operations; no per-user fees | Priced per monthly active user, with MFA, organizations and higher limits often on upper plans | A cross-merchant marketplace grows to a large and spiky customer base; per-user pricing grows with every customer, not with revenue (no platform commission, D15) |
| **Data residency & GDPR** | Runs in our EU region, on our infrastructure; covered by our backups and erasure flow | Customer credentials and profiles held by an additional sub-processor; EU hosting depends on vendor region and plan | NFR-801/802: all personal data in the EU; fewer sub-processors and DPAs to manage; erasure (NFR-804) stays fully under our control |
| **Cloud-agnostic deployment** | Runs on Kubernetes like every other service | Ties us to a vendor; Cognito-type options tie us to one cloud | D75: no lock-in to a cloud or SaaS vendor |
| **Customisation** | Full control: two realms, merchant membership, custom claims, conditional MFA for Owners, event-listener extension, themed pages | Possible through vendor rules/actions, but bounded by vendor features, plan limits and rate limits | Our model is specific: two identity types with the same email allowed (D51), `merchant_id` + role claims, Owner-only MFA, `identity.events` feeding wallets and notifications |
| **Failure domain** | Same region and AZs as the rest of the platform; we control capacity | External dependency with its own SLA and incidents outside our control | Login availability is in line with our own 99.9% target; booking itself never depends on the IdP (local JWKS validation) |
| **Exit cost** | Standard OIDC; data in our own PostgreSQL | Standard OIDC, but password-hash export and migration depend on the vendor | Keeps the option to move later |

**Trade-offs we accept:**

- We operate a clustered, stateful Java service: capacity, upgrades (frequent major releases), security patches, theming and extension builds are our responsibility.
- Security fixes reach us only when *we* upgrade; Keycloak advisories must be tracked and applied promptly.
- No vendor-provided extras out of the box (e.g. bot detection on login, breached-password checks) — covered by our WAF/bot protection and a password-policy extension (Q7 in the architecture doc).

**When we would revisit this decision:**

- The team cannot sustain Keycloak operations (upgrades lagging behind security advisories).
- Requirements appear that a managed IdP covers much better (e.g. enterprise SSO/federation for chains, advanced fraud detection).
- Managed pricing with EU residency becomes cheaper than our operating cost at actual usage.

## Consequences

- No custom code for passwords, MFA, sessions, token signing or key rotation; security fixes come with Keycloak upgrades, which must be kept current.
- The `/auth` zone of our API shrinks: login, MFA, refresh and logout move to Keycloak's standard OIDC endpoints (endpoint catalogue §1).
- Login in the embedded widget uses a popup or redirect, because IdP cookies in third-party iframes are blocked by several browsers. Guest booking in the widget is unaffected.
- Keycloak becomes part of the login path but not the booking path: already-issued tokens keep working during a Keycloak outage (local JWKS validation), until they expire.
- NFR-702's breached-password check needs a Keycloak password-policy extension (identity deep-dive).
- Operating Keycloak (clustering, upgrades, themes, extension builds) is a new operational responsibility.
