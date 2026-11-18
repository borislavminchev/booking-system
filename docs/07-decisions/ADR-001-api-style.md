# ADR-001: Client-facing API style — REST + Server-Sent Events

- **Status:** Accepted
- **Date:** 2026-11-12
- **Decision:** D64

## Context

Clients are the marketplace web/app, the merchant booking page and embeddable widget, and the merchant app. Interactions are request/response (search, slots, hold, checkout, configuration) plus one-way live updates: merchant calendars and booking status must refresh within ≤2 s (NFR-503). There are no external integrators in the MVP. We want the simplest design that meets the requirements and can grow.

## Options

| Option | Pros | Cons |
|---|---|---|
| **REST/JSON + SSE** | One familiar style; HTTP caching for public reads; SSE is plain HTTP, one-way, auto-reconnect with `Last-Event-ID` | SSE needs a fetch-based client to send auth headers |
| REST only, polling | Simplest | Polling every ≤2 s for calendars wastes capacity and battery |
| REST + WebSockets | Bidirectional, low latency | Stateful connections, more infra and auth work for a one-way need |
| GraphQL | Flexible reads for apps | Harder caching, rate limiting and idempotency; overkill for MVP |

## Decision

REST/JSON over HTTPS for all reads and commands; SSE for server → client updates (thin "something changed" events, clients refetch). No webhooks, gRPC or GraphQL in the MVP.

## Consequences

- One style to document, test and secure (NFR-603 negative tests per endpoint).
- SSE connections are long-lived; the architecture phase must place them behind infrastructure that supports streaming and fan-out.
- Webhooks (merchant integrations), a GraphQL BFF or WebSockets can be added later via new ADRs without changing the REST core.
