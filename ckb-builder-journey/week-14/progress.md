# Week 14 Progress

**Date:** 6 - 12 July 2026
**Track:** CKBuilder — Community Keeps Building (Nervos)
**Project:** Even Keel — channel liquidity manager for Fiber Network node operators

---

## Overview

The week Even Keel went from monitoring spine to shipped product: the money path (Phase 2), autopilot and the simulation harness (Phase 3), and the ship phase (Phase 4) — then, mid-ship, the organizers ruled that a live demo during a judging window is mandatory, which reshaped the deliverable from "hosted mock demo" to "live testnet node, rehearsed runbook." By week's end the whole system was rehearsed live (multiple real rebalances settled on Pudge through the product's own executor), the backend was migrated to a Contabo VPS behind a Vercel-hosted dashboard, and the submission form was drafted.

---

## What shipped

### Phase 2 — the money path

- **Planner** (pure core): greedy pairwise selection per the architecture — same-asset partition (UDT never pairs with CKB), candidates ranked by deviation × capacity, amounts sized so neither channel overshoots the target ratio, benefit/fee acceptance as integer cross-multiplication, per-pair cooldowns against fee-burning oscillation. Property tests: bounds always respected, only unhealthy channels touched, budgets inviolable for arbitrary inputs.
- **Executor state machine**: `PLANNED → PRICED → SUBMITTING → CONFIRMING → SETTLED/FAILED`, strictly one action in flight. Every send is preceded by a `dry_run` price on identical params; the action row is written *before* the send RPC so the crash window is bounded; startup reconciliation adopts in-flight payments from `list_payments` by (self-target, amount, time window) or marks them orphan-suspect. Stuck payments block the queue instead of racing it. Fees enter the daily ledger from `get_payment` actuals, never quotes.
- **Advisory flow end-to-end**: proposal card with the exact dry-run fee → operator click → settlement in the action log. A seven-scenario MockNode suite (dry-run-yes-send-fails, stuck payment, crash recovery, transient RPC failure…) drives the real executor; the suite immediately caught a real bug (intent-ID collisions across nodes sharing a database).

### Phase 3 — autopilot + simulation

- **Policy engine**: persisted per-node policy (budgets, thresholds, cooldowns) editable live via `GET/PUT /api/policy`; autopilot is opt-in, default OFF, and every autopilot action records the serialized policy snapshot that authorized it — the audit trail answers "why did it move my money" permanently.
- **24h deterministic simulation** through the *real* executor/store code paths: steady-drain ends the day at 34.3% mean imbalance managed vs 41.7% unmanaged for 0.44 CKB in fees; on oscillating traffic the hysteresis correctly refuses to chase — near-zero spend. Double runs produce byte-identical reports. Property test: for any policy, fees ≤ daily cap AND net imbalance never increases.
- Tried parallel subagent-driven development for the two independent Phase 3 tracks (worktree isolation, strict file-ownership contracts). Mixed result: good architecture came out of it, but session limits cut both runs and integration still needed hands-on finishing — a useful data point on where that workflow helps and where it doesn't.

### Phase 4 — ship, reshaped by the live-demo ruling

- Organizers ruled mid-week: **live demo mandatory** during a judging window; the backend node must be live for that window. Responded by rehearsing the full live flow same-day: Even Keel in real mode against the funded Pudge node — policy tightened via API → planner proposed → priced by the node's real `dry_run` → approved → **settled a real 19.9 CKB circular rebalance** (fee exactly the quote, sink channel landing on the 80.00% target to the basis point). Two more live settlements followed in later rehearsals, including a full practice run of the presenter script.
- **Live mode became the default** (`docker compose up` now expects a real FNN; the deterministic mock is one env var away as the CI/test/simulation environment) — an honest inversion once the live path was proven.
- Ship assets: README's "what's real vs what's simulated" and "the gap this addresses" sections, MIT license, judging-window runbook (`docs/live-demo.md`), spoken presenter script, demo-video shot list, nine-deliverable submission checklist, and the full submission form drafted.
- **Deployment**: FNN node migrated from the laptop to a Contabo VPS (stop → move state → start → archive the source copy so the identity can never run twice), full stack in compose alongside five other backends (port arbitration via compose override), API public for the Vercel-hosted dashboard to proxy against. Judging bring-up is now ~zero: node and backend are always-on.

## Key learnings

- **Requirements change mid-ship; phase discipline absorbs it.** The live-demo ruling landed after the "ship" plan was written. Because the money path had been proven on testnet in week 13's Phase 0 gate, the pivot cost one afternoon (rehearse, write runbook, flip defaults) instead of a rebuild.
- **The state machine earns its keep in the ugly paths.** Everything interesting in Phase 2 was failure handling: the crash window between send and hash-persist, stuck payments blocking the queue, reconciliation by amount+time when the handle is lost. The happy path was the easy 20%.
- **Deterministic simulation is an honesty device, not a toy.** Running the real executor against scripted traffic produces claims that are checkable (byte-identical reports, property-tested invariants) — far stronger demo material than a staged happy path, and the negative result (oscillating traffic → refuse to spend) is the most persuasive chart.
- **Payment-channel node migration is a one-way door**: a channel node's identity must never run in two places (stale state can broadcast an old commitment). Stop, move, start, archive — in that order, always.
- **Upstream fragility is context for design**: FNN issues #1544/#1545 (a lost `ChannelReady` can strand a channel with no local exit path, not even force-close) validated both our is-ready gating and the decision to treat channel-open as the riskiest operation in any mainnet plan.

## Plan for next week

- Finish submission: Vercel↔VPS wiring confirmed, record the demo video from the shot list, fill the hosted/video URLs into the form and the in-repo checklist.
- File the funding-cell race issue against `nervosnetwork/fiber` (concurrent `open_channel` calls race the same live cells; reproduced and documented during the spike) and comment field corroboration on #1544.
- Judging window: run the rehearsed runbook against the always-on VPS backend.
- Optionally, post-submission: a tightly-scoped mainnet pilot (fresh key, expendable funds, paranoid budgets, API auth first).
