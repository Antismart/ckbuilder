# Week 13 Progress

**Date:** 30 June - 5 July 2026
**Track:** CKBuilder — Community Keeps Building (Nervos)
**Project:** Even Keel — channel liquidity manager for Fiber Network node operators

---

## Overview

New project week, targeting the "Gone in 60ms" Fiber Infrastructure Hackathon (Category 3: Liquidity). Fiber channels drift out of balance as payments route through them, a depleted channel silently stops forwarding in one direction, and the FNN README lists "advanced channel liquidity management" as an unchecked TODO — Lightning grew a whole tool category here (lndmanage, charge-lnd, bos rebalance) that Fiber doesn't have yet. Even Keel ([Antismart/evenkeel](https://github.com/Antismart/evenkeel)) monitors channel balance drift and corrects it with circular self-payments, advisory by default, autopilot opt-in and budget-bounded.

The week had two halves: a prep-week spike that put the whole plan behind a kill-criterion gate (if circular routes are unfindable on testnet, pivot to advisory-only before writing any code), and — once the gate went green — the Phase 1 build: the full monitoring spine from a pure Rust decision core to a live dashboard, packaged as a one-command compose stack.

---

## What shipped

### Phase 0 — the testnet gate: GREEN

Stood up FNN v0.8.1 on Pudge testnet and proved the core money primitive end to end:

- **Real circular self-payment settled.** `send_payment(target_pubkey: self, keysend: true, allow_self_payment: true)` moved 100 CKB out of one channel and back into another. The `dry_run: true` quote said 0.1 CKB fee; `get_payment` reported the actual fee as **exactly** 0.1 CKB. Principal untouched — the node's total changed by precisely the routing fee. That asymmetry (success costs a known fee, failure costs nothing) is the entire safety story of the tool.
- **Built FNN from source** — Docker Hub/GHCR publishing for `nervos/fiber` only starts at 0.9.0-rc, so no 0.8.x image exists. The spike tooling (`ops/spike/`) builds v0.8.1 from the upstream Dockerfile automatically.
- **Solved the RPC bind restriction** — v0.8.x refuses "public" RPC binds (including `0.0.0.0`) without biscuit auth, but RFC1918 addresses pass its `is_public_addr()` check. A static compose-network IP with a host-loopback-only port mapping keeps the spending-capable RPC private without auth machinery.
- **Documented the testnet friction honestly** in `docs/spike-notes.md`: a fragmented public graph (most well-connected nodes announce unreachable private IPs), peers that accept TCP but never complete the Fiber `Init` handshake, the inbound-liquidity bootstrap (fresh channels can't be a return leg until a keysend push gives the counterparty balance toward you), and a funding-transaction race when opening two channels concurrently against one wallet cell.

### Phase 1 — the monitoring spine, green end to end

Four-crate Rust workspace plus a Nuxt dashboard, each layer committed only when tests and clippy were clean:

- **`evenkeel-core`** — pure decision core, no I/O. Usable-liquidity math (`local − offered_tlc`, netting out in-flight TLCs) in integer-only `u128` Shannons and basis points; five-class health classification (depleted / depleting / healthy / filling / saturated) where drift — an exact integer least-squares slope in bp/hour — catches a channel at 35% and falling before it ever hits the depleted threshold. Six proptest suites pin the invariants: classification monotone in ratio, thresholds bind exactly, ratio math total across the full `u128` range.
- **`evenkeel-node`** — a `FiberRpc` trait with two implementations: `RealNode` (typed structs mirroring the v0.8.1 wire format, serde fixtures shaped from live spike responses) and `MockNode`, a first-class artifact with deterministic per-tick balance scripts and fault injection. One test polls the draining mock through the real health engine and watches it classify `depleting` — the whole decision path verified with no network.
- **`evenkeel-store`** — Postgres snapshot time-series with `NUMERIC(39,0)` money columns crossed as decimal strings, so `u128::MAX` survives storage exactly. Committed sqlx offline data means builds need no database.
- **`evenkeel-server`** — one serialized control loop (poll → persist → classify → publish), Prometheus gauges for ratio/drift/staleness, and a REST API that degrades to stale-data-with-a-banner when the node RPC dies instead of taking the dashboard down.
- **Dashboard** — channel cards with usable-ratio meters, drift sparklines with the 20%/80% classification bounds drawn in, and health badges that never rely on color alone.
- **`docker compose up`** — brings up Postgres + server + dashboard against a scripted mock scenario (healthy / draining / saturated) with no Fiber node and no tokens; the real node is opt-in behind a `testnet` profile.

---

## Key learnings

- **Gate before build.** Putting one real testnet payment before any code turned "will rebalancing even work on Pudge?" from a mid-hackathon crisis risk into a settled fact with evidence. The pivot plan (advisory-only) was written down before the answer was known, so red would have been a decision, not a scramble.
- **The failure modes are the design.** Nearly everything the spike surfaced fed straight into architecture decisions: the funding-cell race independently validates the serialized one-action-at-a-time executor; sparse testnet routing validates mock-first development; the exact-fee settlement validates pricing every action with `dry_run` before commitment.
- **Integer money is cheap on day one, expensive to retrofit.** `u128` Shannons end to end, basis points in every decision path, floats only at the display edge (and even the dashboard formats balances with `BigInt`). The property test that hammers ratio math across the full `u128` range found its overflow path immediately.
- **Supply-chain policies now bite in containers.** A newer pnpm inside the image enforced a minimum-release-age policy and rejected day-old lockfile entries that the host's older pnpm had accepted — pinning `packageManager` in `package.json` makes host and container agree.

---

## Plan for next week

- Flip Even Keel to Phase 2 — the money path: greedy pairwise rebalance planner (same-asset only, benefit/fee ranking, hysteresis), the executor state machine (PLANNED → PRICED → SUBMITTING → CONFIRMING → terminal, with crash reconciliation via `list_payments`), and the daily fee ledger fed by actual fees from `get_payment`.
- Advisory flow end to end in the dashboard: propose → show dry-run fee → operator click → settle → action log.
- MockNode scenario suite for the ugly paths: dry-run-says-yes-then-send-fails, stuck payments, RPC timeouts, crash-recovery reconciliation.
