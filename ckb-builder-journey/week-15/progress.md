# Week 15 Progress

**Date:** 23 - 29 July 2026
**Track:** CKBuilder — Community Keeps Building (Nervos)
**Project:** Cellora — managed indexing and query layer for CKB

---

## Overview

Back on Cellora, this week was a security-hardening pass on the live backend rather than new features. The question was blunt: is this actually production-ready? To answer it honestly I ran adversarial multi-agent code reviews over the Rust backend — a fan-out of specialist reviewers, each finding independently verified by a skeptic agent trying to *refute* it, then ranked by severity with a fix plan. Two rounds surfaced twelve real, reachable defects; all twelve are now fixed, tested, and pushed, with the build, clippy, and fmt gates green.

---

## What shipped

### First review pass — five high/medium security bugs

The first fan-out (reviewers across reorg handling, auth, rate limiting, SQL/pagination, concurrency, API input, and panic vectors) raised ten findings; adversarial verification confirmed nine and refuted one. The five that mattered got fixed:

- **Deep-reorg data loss (high).** The poller resumed indexing at the *poll* height instead of the true common ancestor, so any reorg deeper than one block permanently deleted and never re-fetched the blocks in between — a silent correctness bug in the core indexer, the exact case the reorg subsystem exists to handle. Threaded the ancestor height back through the rollback path so it resumes at `ancestor + 1`.
- **Webhook SSRF (high).** Any authenticated user could register `http://169.254.169.254/…` or a loopback URL and have the dispatcher POST to it. Added URL validation (scheme + non-public-IP rejection) and disabled redirect following.
- **Fake webhook signature (high).** The `x-webhook-signature` header shipped the *raw secret* verbatim on every request. Replaced it with a real HMAC-SHA256 over `{timestamp}.{body}`.
- **OAuth login CSRF (medium).** The `state` token lived only in Redis with no browser binding, allowing session fixation. Bound it to an HttpOnly cookie the callback must match.
- **GraphQL amplification (medium).** No depth/complexity limit, so one request could fan aliased `cells` fields into dozens of heavy DB queries for one rate-limit token. Added depth and complexity bounds.

### Second review pass — scrutinizing the fixes, then seven more

The second fan-out was told exactly what the first pass had changed, and pointed at verifying those fixes rather than re-reporting them. It confirmed the fixes held — and found seven further defects, all fixed:

- **Unbounded LIFO Redis metrics queue (high).** Request logs were `LPUSH`ed and `LPOP`ped from the same end (LIFO, so old logs starved) with no cap — an unbounded queue on the Redis that also backs rate-limiting and readiness, a memory-exhaustion DoS. Made it FIFO, added an `LTRIM` cap, and drain-to-empty per tick.
- **Session-cookie parser bug (medium).** A value-less cookie segment aborted the whole scan via a stray `?`, so a valid session cookie later in the header read as absent (spurious 401). Notably, the reviewers caught that the OAuth cookie parser I *added* last pass used the correct pattern while the older session parser still had the bug.
- **Reorg depth off-by-one (medium)**, plus four low-severity items: SSRF allowlist gaps (`0.0.0.0/8`, `240/4`), a DNS-rebinding TOCTOU in the SSRF guard (fixed by pinning the connection to the vetted IP), the webhook dispatcher lacking a shutdown token, and a reorg tight-loop on an inconsistent node.

The seven fixes were themselves applied by a **multi-agent fix workflow**: four specialist agents (indexer/reorg, webhook security, Redis reliability, auth), each owning a disjoint set of files so they could run in parallel without stepping on each other, followed by an integration verifier that compiled everything and would have repaired any breakage. Ten new DB-free unit tests, all passing.

---

## Key learnings

- **Adversarial verification earns its keep.** Making a second agent try to *refute* each finding — defaulting to "not a bug" unless it can trace a reachable path — is what separates a real defect list from a pile of plausible-sounding noise. Across both passes it refuted only one finding, but that discipline is why I trusted the other sixteen enough to fix them.
- **Every fix creates the next finding's surface.** Closing the SSRF hole exposed a residual DNS-rebinding TOCTOU; the OAuth cookie fix I wrote in pass one became the reference implementation that revealed the *older* session parser had the same latent bug. Reviewing the fixes is as valuable as the original review.
- **Partition parallel agents by file ownership, not by task.** Letting four fix-agents run concurrently only works if no two can touch the same file. Assigning disjoint file sets up front turned a merge-conflict risk into a clean parallel run.
- **The scariest bug was the quietest.** The deep-reorg data loss didn't throw, didn't need an attacker, and would silently corrupt the exact dataset the product sells — a good reminder that "production-ready" is about correctness under adverse conditions, not the happy path.

---

## Plan for next week

- Clear the GitHub Actions billing block so CI actually runs: the integration suite (which needs Docker/testcontainers) has still never executed end to end, so "build + clippy + unit tests green locally" is as far as verification currently reaches.
- Close out the remaining known Cellora gaps from the production-readiness review: decide fail-open vs. fail-closed for the rate limiter under a Redis outage, build the webhook retry queue (the dispatcher is still at-most-once), and write the deployment runbook.
