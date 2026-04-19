# Week 4 Progress

**Date:** 13 - 19 April 2026
**Track:** CKBuilder — Community Keeps Building (Nervos)

---

## Overview

This week marked the shift from learning into building. With the CKB Academy and CCC Playground complete, I focused on scoping, validating and kicking off the capstone project. I finished the week with the Week 1 roadmap of the capstone shipped: a working Rust service that connects to a CKB node, ingests blocks and stores them in PostgreSQL.

---

## Capstone planning

### Idea
A managed CKB indexer as a service. A clean REST and GraphQL API that lets developers query on-chain cell data without running their own node or indexer. Think of it as what Alchemy or Infura are for Ethereum, but CKB-native.

### Research done
Before committing to the idea I looked at what already exists in the ecosystem:

- The CKB node has a built-in indexer since v0.106.0, but it's raw JSON-RPC and requires every developer to run and maintain their own node
- Mercury was the closest thing to a managed service, but it hasn't had an update since April 2024 and still requires self-hosting
- GetBlock and Ankr provide raw RPC node access, not a clean developer-friendly data API

The conclusion: there is no polished, managed service where a developer can sign up, get an API key and start querying CKB cell data without infrastructure setup. That's the gap this project fills.

### Validation
I ran the idea by Neon who thought it sounded interesting and suggested I also check with Matt and Retric for deeper insight, especially around Mercury.

Retric gave strong feedback:
- Confirmed this is valuable infrastructure for the ecosystem
- Raised a concern about whether a capstone heavy on web2 infrastructure would still be CKB-native enough — concluded it is, since the project involves running a CKB node, handling cells and transactions, dealing with reorgs and understanding the chain's data structures directly
- Advised that the product be designed from a real developer's perspective, collecting actual pain points from the community rather than guessing at the API surface

This last point shaped the approach for Week 2 onwards — rather than locking in the API shape early, the first priority is to talk to CKB builders and pull real needs from the community.

---

## Architecture

I spent a chunk of the week designing the system properly before writing any code. The goal was to plan for a real product, not a toy, so the architecture assumes multi-tenant SaaS traffic, reorgs, horizontal scaling and production observability.

### Core principles
- Reliability over features — missed blocks are bugs
- Horizontal scalability from day one
- Observability first — every request, block and error is traced and metered
- Security by default — hashed API keys, TLS everywhere, WAF at the edge
- Deterministic data — the database is always reconstructable from the chain

### System layers
1. **Client layer** — DApps, wallets, explorers, SDKs
2. **Edge / CDN** — Cloudflare for TLS, DDoS protection, caching
3. **API gateway** — Rust + Axum serving REST and GraphQL, with auth and rate limiting
4. **Service layer** — separate query service (stateless, horizontally scalable) and indexer service (stateful, singleton with standby)
5. **Data layer** — PostgreSQL primary + read replicas, Redis for caching and rate limit buckets
6. **CKB node layer** — self-hosted full nodes on mainnet and testnet

### Key design decisions
- Query and indexer are separate services so they scale independently
- PostgreSQL tables will be partitioned by block range for performance as the chain grows
- Reorgs are handled explicitly by rolling back affected blocks and re-indexing from the divergence point — this is where most toy indexers fall over
- API keys are hashed with Argon2, never stored in plaintext
- Rate limiting is Redis-backed token buckets, per API key, separate buckets for REST and GraphQL

### Incremental delivery plan
The project is broken into 7 weekly milestones. Each week ends with something working, tested and deployable. No "build everything at once" — scope creep is the biggest risk with a project this size.

The full roadmap in order:
1. Foundation and block ingestion
2. REST API
3. Authentication, rate limiting, and GraphQL
4. Reorg handling and observability
5. Dashboard and developer experience
6. Webhooks and subscriptions
7. Billing and production readiness


---

## Week 1 of capstone — shipped

With planning done, I worked through the Week 1 scope of the roadmap:

### What was built
- Cargo workspace with `indexer`, `db`, and `common` crates
- `docker-compose.yml` spinning up PostgreSQL, Redis and a CKB dev node
- Initial SQL migrations for `blocks`, `cells` and `transactions` tables
- CKB JSON-RPC client wrapper in the `common` crate
- Block poller in the `indexer` crate:
    - Polls the CKB node every 2 seconds
    - Tracks the last indexed block
    - Parses block data into cells and transactions
    - Writes each block atomically in a single database transaction
- Graceful shutdown handling with `tokio::signal`
- Structured logging throughout using `tracing`

### Quality standards applied
- No `unwrap()` or `expect()` in non-test code — all errors flow through `thiserror`-derived types
- Every public function is documented
- SQL queries use `sqlx::query!` macros for compile-time checking
- Integration test spins up the docker-compose stack and verifies blocks are indexed end to end
- Conventional commit messages throughout

### Explicitly deferred to later weeks
- No API layer yet (Week 2)
- No reorg handling (Week 4 — happy path only for now)
- No authentication (Week 3)
- No Redis integration (Week 3)

---

## Key learnings
- Doing the research on existing solutions before committing saved me from building something redundant
- Getting validation from Retric before going deep gave me confidence the gap is real, plus the steering away from premature API design was valuable
- Writing the architecture first, then constraining the build into weekly slices, made it much easier to start coding without getting overwhelmed
- The workspace structure in Rust (multiple crates under one `Cargo.toml`) is paying off already — clean separation between the indexer, API and shared types

---

## Blockers
None. Planning took longer than initially expected but that was the right call — shipping the wrong thing fast would be worse than shipping the right thing carefully.

---

## Plan for next week
- Reach out to CKB builders in the community to gather real pain points around on-chain data access
- Start Week 2 of the capstone roadmap: REST API
    - Build the `api` crate with Axum
    - Implement health, blocks, cells and stats endpoints
    - Cursor-based pagination
    - Consistent JSON error format
    - Generate OpenAPI spec
