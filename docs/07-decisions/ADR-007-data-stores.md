# ADR-007: Data stores — PostgreSQL per service, Redis, search on PostgreSQL

- **Status:** Accepted
- **Date:** 2026-11-15
- **Decision:** D71

## Context

Holds, bookings and money need linearizable writes, range-overlap enforcement and multi-row ACID transactions (NFR-101–105); RPO≈0 with synchronous cross-AZ replication (NFR-204); ~1B+ ledger rows over 7 years (NFR-405). Read models need fast caching (slots ≤300 ms at ~10k/s), rate limiting, and SSE pub/sub. Marketplace search covers ~10k merchants with keyword + location and may lag 5 min (NFR-504).

## Options

| Option | Pros | Cons |
|---|---|---|
| **PostgreSQL + Redis** | Mature ACID, exclusion constraints on ranges, PostGIS + full-text for search; managed multi-AZ offerings everywhere; Redis for ephemeral fast data | Single-primary write scaling per cluster |
| PostgreSQL + Redis + OpenSearch | Richer search relevance | One more cluster to run for ~10k merchants |
| Distributed SQL (CockroachDB, Spanner-like) | Horizontal write scale | Cost, latency of consensus writes, less mature range-exclusion support |

## Decision

- **Database per service on PostgreSQL**; Booking & Payments on a dedicated cluster, others may share a cluster as separate databases at launch.
- Every cluster: primary + synchronous standby in another AZ, PITR backups copied to a second EU region.
- **Redis** (replicated, multi-AZ) for gateway rate limits and hold caps, Availability slot cache, SSE pub/sub and replay streams. Never a source of truth.
- **Search** in the Discovery service's PostgreSQL with PostGIS + full-text; a dedicated search engine is added only when relevance or load requires it.

## Consequences

- No-double-booking can be enforced declaratively (GiST exclusion constraint) inside the same transaction as the ledger.
- Write capacity of Booking & Payments is bounded by one primary until a sharding plan exists (architecture Q4).
- Large tables (bookings, events, ledger entries) are partitioned by time to keep 7 years online.
