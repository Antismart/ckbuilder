# Week 5 Progress

**Date:** 20 – 26 April 2026
**Track:** CKBuilder — Community Keeps Building (Nervos)

---

## Overview

Two tracks in parallel this week: community outreach to surface real developer pain points before the API surface hardens, and shipping Week 2 of the capstone (the REST API) so there's a working surface to hand those developers.

---

## What I worked on

### Community outreach

- Developer survey. A 21-question form covering how CKB builders access chain data today, which query patterns matter, preferred interfaces, and rough willingness to pay. Structured to produce signal, not demographics.
- Forum post on talk.nervos.org. Introduced the project to the wider Nervos community and seeded the survey. Framed as a request for input rather than an announcement, aimed at builders who read the forum but don't live in chat.
- Issue on the CKBuilder-projects repo. Submitted the project through the CKBuilders review channel with specific design and product-fit questions. A substantive reply from the CKB team surfaced a thread on verifiable data / trust model that changed a design decision the same week.

### Week 2 capstone — REST API shipped

- New `api` crate on Axum with tower-http middleware (trace, request-id, timeout, panic catcher) and an integration-test harness against a testcontainers Postgres, no sockets opened.
- Endpoints: `/v1/health`, `/v1/health/ready`, `/v1/blocks/latest`, `/v1/blocks/:number`, `/v1/cells` (opaque base64url cursor pagination, `lock_hash` / `type_hash` / `is_live` / `include_data`), `/v1/stats` backed by an arc-swap tip cache refreshed on a background task, `/v1/openapi.json`.
- OpenAPI 3 spec generated from the code via utoipa, committed to `docs/openapi.json`, and drift-checked by a test that fails on mismatch.
- `X-Indexer-Tip` and `X-Indexer-Tip-Stale` response headers on every 2xx so clients can compute their own freshness.
- Cell responses carry `block_hash` alongside `block_number` — direct consequence of the CKB-team feedback on verifiable data. Costs almost nothing to add now, would have been a breaking change later.
- License switched from Apache-2.0 to FSL-1.1-ALv2 so the public repo stays a community asset without inviting a fork-and-compete play.
- 52 tests green; `cargo clippy --workspace -- -D warnings` clean.

---

## Key learnings

- Writing the survey forced specificity about assumptions I was making about the API — several questions exist because I realised I didn't actually know the answer.
- Splitting outreach across survey + forum + issue covers different developer habits: some respond to forms, some to long-form discussion, some only see things in their GitHub feed.
- The trust-model feedback on the CKBuilders issue landed mid-week and changed slice 3 of the REST work. The cost of rolling `block_hash` into cell responses while the code was fresh was nearly zero; retrofitting it later would have been a breaking API change. Fast feedback loops pay compound interest.

---

## Blockers

- None on the work itself. Survey responses will take time — Week 3's design (auth, rate limiting, tiers) leans harder on that signal than Week 2's did.

---

## Plan for next week

- Collect and review survey / forum / issue responses so the input is informing Week 3 rather than being post-hoc justification.
- Begin Week 3: API-key auth (Argon2), per-key Redis rate limiting with separate REST / GraphQL buckets, and the GraphQL surface on async-graphql.
- Post the follow-up on the verifiable-data thread with a concrete design stance — trust-model section in the architecture doc, proof-passthrough endpoint sketched for Week 4.
