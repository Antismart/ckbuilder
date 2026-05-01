# Week 6 Progress

**Date:** 27 April – 1 May 2026
**Track:** CKBuilder — Community Keeps Building (Nervos)

---

## Overview

Heavy capstone week. Wrapped the auth, rate limiting, and GraphQL slice of the roadmap and pushed through most of the next one (reorg handling and observability) so the indexer can survive a chain reorg without operator intervention and the whole stack is legible from the outside. Five new ADRs landed alongside the code. Beta-outreach material was drafted in parallel so the API isn't hardened in a vacuum.

---

## What I worked on

### Capstone — Auth, rate limiting, GraphQL

- `api_keys` schema with Argon2id hashing, a `tier` field for future plan gating, and an admin CLI for issuing and revoking keys. Plaintext keys are shown once on creation and never stored.
- Bearer-token authentication on every `/v1/*` data endpoint, fronted by a verification cache so the Argon2 cost is paid once per key per process, not per request.
- Redis token-bucket rate limiter, per-key, returning `429` with a `Retry-After` header when a bucket drains. Limits are configured per tier.
- GraphQL endpoint at `/graphql` on async-graphql, sharing the auth layer with REST and carrying its own per-surface rate-limit bucket so a heavy GraphQL client can't starve REST callers (or vice versa).
- k6 load test in `tests/load/` that exercises the rate limiter end-to-end — confirms the 429 path, the recovery curve, and that REST and GraphQL buckets are actually independent.

### Capstone — Reorg handling and observability

- Reorg detection by walking parent hashes back from the new tip until the fork point is found, then a transactional rollback of every affected block, cell, and transaction in a single Postgres transaction. The indexer no-ops on the happy path and only pays the walk on a hash mismatch.
- New `reorg_log` audit table that captures fork point, depth, and affected row counts for every reorg the indexer handles. Cheap to write, invaluable for incident review.
- Reorg test suite that fabricates competing chains and asserts the database ends up consistent with the winning fork.
- Prometheus metrics on both the indexer and the API — shared rate-limit counters, request counters, and the indexer's lag-from-tip — exposed on a `/metrics` endpoint.
- Optional OpenTelemetry OTLP tracing layer behind a feature flag so traces ship to a collector when one is configured and stay out of the way when one isn't.
- `/v1/health/ready` expanded to actively probe Redis and the CKB node, not just check that the indexer process is up. The previous version would have answered "ready" while the upstream node was unreachable.
- Well-known script registry — cells are tagged with `lock_kind` and `type_kind` at index time so common queries ("all sUDT cells", "all multisig locks") don't require clients to know hashes by heart. Came directly out of survey signal.
- `/v1/proofs/:tx_hash` passthrough that exposes `get_transaction_proof` from the underlying CKB node, the first concrete piece of the verifiable-data trust model from last week's forum thread.


---

## Key learnings

- Reorg handling is one of those features where the test suite has to be more imaginative than the production path. The interesting bugs aren't in detecting a reorg — they're in cleaning up state that other queries are mid-flight against. The transactional rollback is the only thing keeping the cell view consistent.
- Splitting REST and GraphQL into separate rate-limit buckets isn't obvious until you imagine a single noisy GraphQL client. The shared-Redis design made this a one-line change at the limiter, which is the kind of thing you only get if you wire the abstraction in early.
- ADRs work best when written *with* the code, not after it. Three of the five this week genuinely changed the implementation between draft and merge — by the time I'd argued the alternatives in prose, the obvious objection became obvious.
- The verification cache for API keys was the single biggest p99 win of the week. Argon2 is intentionally expensive; paying it per request was untenable, paying it per key per process is fine.

---

## Blockers

- None on the engineering. Two deliverables didn't get committed in time and roll into next week:
  - The new docs (`docs/api.md`, ADRs 0002–0006, `docs/research/`, the architecture overview edits, and the GraphQL section tweak) are uncommitted on disk and need a docs-only commit.
  - The Grafana dashboard JSON for the new Prometheus metrics isn't yet checked into `ops/dashboards/` — it was the last observability deliverable and slipped.

---

## Plan for next week

- Land the outstanding docs commit and the Grafana dashboard JSON first thing — closing out the observability slice cleanly before opening the next one.
- Begin the next capstone slice: developer dashboard (key management, usage graphs, request log) and the DX surface — typed client, quickstart, example apps. Real consumers of the API now exist on paper; this is what they actually touch.
- Post the follow-up on the verifiable-data forum thread once the proofs endpoint and trust-model ADR are linkable from the public docs.
- Start collecting survey responses against the new outreach drafts — the dashboard scope should be informed by what builders say they want to see, not just what's easy to render.
