# Week 17 Progress

**Date:** 6 - 12 August 2026
**Track:** CKBuilder — Community Keeps Building (Nervos)
**Project:** Cellora — managed indexing and query layer for CKB

---

## Overview

A week back on Cellora that split cleanly into polish and a real strategic decision. First I reworked the landing page copy to cut the AI-marketing slop. Then I went deep on the question the community's design review kept circling: is a trusted hosted indexer the right shape, and how far toward verifiable or decentralized should it go. I answered it by committing to a verifiable-completeness and reproducibility track (no token), and writing that up as an architecture decision so it is on the record rather than floating as an aspiration.

---

## What shipped

### Landing page copy, de-slopped

Ran a strategist to section-specialists to editor workflow over the landing page to strip generic AI-marketing copy. The core fix was repetition: the "stop running your own CKB infra" pitch was restated about four times across the hero, value props, and features, so I collapsed it to one sharp positioning line and made every other section add new information. The pass also removed fake trust signals (an "All systems normal" status badge with no status page behind it, and seventeen placeholder links that all pointed at "#"), and caught a fabricated code sample that imported from a `@cellora/sdk` package that does not exist, replacing it with a real `fetch()` call against the actual API. What is left reads like an engineer wrote it.

### Trust model and the decentralization question (the real work)

I went back through the Nervos Talk design-review thread (matt_ckb, ArthurZhang, phroi). The recurring theme is that a trusted indexer is a trust surface whether you name it or not, and this community wants verifiability. First I confirmed the near-term commitments from that thread had actually shipped: the trust-model ADR, block-hash annotations on every record, the `/v1/proofs/:tx_hash` inclusion-proof passthrough, well-known script tagging, and first-class reorg handling. Executing on the feedback rather than just nodding at it is the strongest signal that the design is sound.

The open concern was phroi's completeness problem: verifying a result the API returns is a different guarantee from proving nothing matching was omitted. That opened the bigger question of whether to emulate The Graph as a decentralized protocol. I separated it into levels: verifiable responses, then reproducible multi-operator, then a full staked-token protocol. The decision was to commit to the first two and explicitly decline the third:

- **Verifiable completeness plus reproducibility (committed):** a deterministic, content-addressed state root (a proof-of-indexing) over the fixed schema, plus authenticated-index coverage proofs so a filtered query can prove it returned exactly the matching set relative to that root. Because CKB's schema is fixed and its cell model deterministic, independent operators should reproduce the identical root at the same tip, which is the "independent parties reproducing the same canonical view" idea phroi pointed at.
- **Token protocol (declined):** no staking, delegation, curation, query-fee token, or governance. The economics need query volume and token liquidity that do not exist yet, and the overhead would swamp the product.

I captured all of this as an ADR and a Phase-2 roadmap track, which makes determinism a hard design constraint going forward, and drafted a reply to phroi that engages the reproducibility direction and asks to align on the state-root commitment format.

---

## Key learnings

- **The honest answer to "is a trusted service acceptable" is not decentralization theater, it is trust-minimization stated plainly.** Verifiable escape hatches plus reproducibility, and a clear sentence about what you do not prove, beats a token network you cannot sustain. CKB makes this cheaper than it was for The Graph: a fixed schema and a deterministic cell model mean there is no arbitrary mapping logic to diverge, so a proof-of-indexing is almost free, whereas The Graph had to invent one to paper over exactly that divergence.
- **A written, deliberate "no" is a credibility move.** Declining the full token protocol in the ADR, with the reasoning, reads better to a skeptical audience than a hand-wavy "someday decentralized." Owning the limit is more persuasive than hiding it.
- **Copy slop is a trust signal in the same family as a weak trust model.** A fake "All systems normal" badge and dead placeholder links read as AI-generated filler to a technical audience the same way buzzwords do. Honesty in the marketing should mirror honesty in the architecture.

---

## Plan for next week

- Post the reply to phroi and open the state-root commitment-format conversation with them.
- Turn the Phase-2 proof-of-indexing primitive into a concrete design: what gets hashed, at what tip granularity, and which authenticated structure keeps coverage proofs cheap for the real query shapes. This is the keystone that everything above it depends on.
- Standing backend items, still open: clear the GitHub Actions billing block so CI runs end to end, decide fail-open vs fail-closed for the rate limiter, build the webhook retry queue, and write the deployment runbook.
