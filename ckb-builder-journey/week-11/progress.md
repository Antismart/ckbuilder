# Week 11 Progress

**Date:** 23 - 29 June 2026
**Track:** CKBuilder — Community Keeps Building (Nervos)
**Project:** Cellora — managed indexing and query layer for CKB

---

## Overview

With the stack live on the VPS and the frontend on Vercel, this week shifted from shipping features to hardening the build and release pipeline. We ran a production-readiness review of the backend, then closed the two biggest operational gaps it surfaced: there was no container image and no continuous integration. We added a reproducible multi-stage Docker build and a GitHub Actions workflow that lints, tests, and builds the image on every push.

---

## What shipped

### cellora — Containerization and CI/CD

- **Multi-stage Dockerfile:** A `rust:1.82-bookworm` builder compiles both binaries (`cellora-api` and `cellora-indexer`) and a `debian:bookworm-slim` runtime ships them in one image. The build is fully offline — SQLx compile-time query checking reads the committed `.sqlx/` cache, so no database is needed to build. BuildKit cache mounts persist the cargo registry and target directory for fast incremental rebuilds, and the container runs as a non-root user. Because every TLS client is rustls, the runtime image needs only `ca-certificates` — no OpenSSL.
- **One image, two services:** Rather than two images, the single image carries both binaries on `PATH` and defaults to the API; the indexer is selected by overriding the container command. Same artifact, different command in compose or Kubernetes.
- **`.dockerignore`:** Keeps `target/`, `.git/`, docs, and secrets out of the build context while preserving `migrations/` and the `.sqlx/` cache that the build depends on.
- **GitHub Actions CI:** A `lint-and-test` job runs `rustfmt --check`, `clippy --workspace --all-targets`, and `cargo test --workspace` against a live Postgres service container; a `docker-build` job builds the production image offline. The two jobs together also act as a drift detector for the query cache.

### Backend production-readiness review

Audited the backend against a "would I trust this in production" bar. The encouraging findings: no `unwrap`/`expect`/`panic` in non-test code (enforced by `deny` lints), compile-time-checked SQL, Argon2-hashed credentials, atomic reorg rollback with an audit log, and Prometheus plus OpenTelemetry wired in. The honest gaps logged for follow-up: the rate limiter currently fails open if Redis is unreachable, webhook deliveries are not yet retried, and there is no deployment runbook.

### Docs cleanup

Earlier in the period we corrected the testing guide to match the single-network backend and removed stale multi-network and pricing references that no longer reflect the product.

---

## Key learnings

- **Offline image builds vs. test-time query checking:** The committed `.sqlx/` cache covers library and binary code, so the image builds without a database. But the integration tests carry their own `query!` macros that are not in the cache, so CI needs a live Postgres at compile time for the test targets. Splitting this into an offline image build plus a live-DB test job means the image build doubles as a check that the query cache has not drifted from the code.
- **Self-provisioning tests simplify CI:** Because the integration suites spin up their own throwaway Postgres and Redis via `testcontainers`, the CI runner only needs Docker available — there is no separate fixture orchestration to maintain.
- **rustls keeps the runtime slim:** Choosing rustls over native-tls across reqwest, SQLx, and the OTLP exporter means the runtime image avoids an OpenSSL dependency entirely, which keeps the slim base genuinely small and the dependency surface smaller.

---

## Plan for next week

- Resolve the GitHub Actions billing block so the workflow actually executes, then fix anything the first real run surfaces.
- Decide fail-open vs. fail-closed for the rate limiter under a Redis outage and record the decision as an ADR.
- Implement webhook delivery retries with exponential backoff so failed deliveries are no longer silently dropped.
- Write `docs/deployment.md` and an operations runbook capturing the VPS deploy and recovery steps.
