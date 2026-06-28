# Week 11 Progress

**Date:** 11 - 16 June 2026
**Track:** CKBuilder — Community Keeps Building (Nervos)
**Project:** ckb-crosschain-verification — trustless verification of CKB state inside other chains

---

## Overview

This week was a design week on a new project alongside Cellora. The premise: a contract on Ethereum (or Solana, or a Cosmos chain) cannot answer "did transaction T happen on CKB?" without trusting a bridge multisig, an oracle, or a centralized API. CKB's FlyClient light client (RFC 44/45) already makes that verifiable on phones and in browsers, but nobody has ported that verification *into another chain's execution environment*. The output of the week is a full engineering design document for an SDK and relayer that does exactly that, published to Nervos Talk for community review and sharpened by two rounds of feedback folded back into the design.

---

## What shipped

### ckb-crosschain-verification — engineering design document

Wrote the repository README and a ~500-line design doc covering the protocol end to end:

- **Problem and goals.** A destination-chain contract should be able to verify, trustlessly: (G1) a CKB header is part of the heaviest chain, (G2) a transaction is included in a verified block, and (G3) a cell is an output of a verified transaction — at a cost target under ~500k gas per verified message on EVM, with no committee, multisig, or governance key in the verification path. Safety depends on no one; liveness depends on at least one honest relayer existing.
- **Primitives.** Documented the pieces we build on: the CKB header chain and Eaglesong PoW, the chain root MMR (RFC 44), FlyClient difficulty-weighted sampling (RFC 45), and the CBMT transaction/cell commitment path.
- **Architecture.** Four components — a Rust relayer (header tailer, MMR maintainer, proof builder, per-chain submitter), a ZK proving service for the EVM path, on-chain verifier contracts per destination chain, and a TypeScript + Rust client SDK with a two-call developer experience (`proveTransaction`, `submitAndVerify`).
- **Core protocol.** Specified the heavy tip-update flow (sampled PoW + MMR-root transition + cumulative-difficulty reorg rule) and the cheap, frequent inclusion-proof flow.
- **ZK layer.** Compared SP1 vs RISC Zero vs a custom Halo2/Plonky3 circuit and chose **SP1 first**, because the zkVM guest can reuse `ckb-light-client` verification logic nearly verbatim as a `no_std` crate. Wrote the guest program spec — the linchpin is deriving sample heights via Fiat–Shamir from the new tip hash so a prover cannot grind which heights it proves.
- **Build plan.** A staged plan from Phase 0 de-risking spikes (Eaglesong cycles in SP1, blake2b inclusion gas on EVM, incremental MMR maintenance against mainnet) through a Base-testnet vertical slice, production hardening, and a second native runtime (Solana) to prove the code-reuse bet — each phase with explicit kill criteria.

### Published for community review

Posted the design as **"Design notes: verifying CKB state inside other chains (EVM-first, built on RFC 44/45)"** on Nervos Talk ([thread](https://talk.nervos.org/t/design-notes-verifying-ckb-state-inside-other-chains-evm-first-built-on-rfc-44-45/10372)). The write-up framed the split between rare, expensive zkVM-backed tip updates and cheap, frequent (~60–100k gas) inclusion proofs, the Fiat–Shamir trick that makes RFC 45 sampling non-interactive on-chain, and four explicit asks for the community: sampling parameterization, cell-liveness proofs, Eaglesong proving cost, and whether anyone actually needs this at today's volumes.

### External review folded into the design

- **Asset trust taxonomy (credit: Neon).** Corrected an early framing error: the SDK removes bridge trust on the CKB→destination leg, but it does nothing about custody trust *upstream* of CKB. CKB-native xUDT and Bitcoin-issued RGB++ assets are trustless end to end with the SDK; custodially wrapped BTC (e.g. ccBTC) is not. Committed the project to claim discipline and to a flagship demo using a genuinely-trustless asset.
- **Checkpoint-binding soundness (credit: T_Silva, Chiral — CKB→Cardano).** A reply on the Nervos Talk thread named a soundness class any checkpoint-amortized light client shares: membership against a relayer-maintained checkpoint is insufficient on its own, because a header valid under an earlier checkpoint can be replayed after the checkpoint advances. Folded the fix into a new security section — a mandatory two-sided height bound (`min_confirmations ≤ tip − height ≤ W`), verification only against the live canonical MMR root, and cumulative-work-only advancement from a single genesis anchor — and agreed with them to treat CKB-side Eaglesong proving as shared infrastructure across our different target chains.

---

## Key learnings

- **The reusable crown jewel is the verification crate.** Picking a Rust zkVM (SP1) over a hand-written circuit means the consensus-critical verification logic is one `no_std` crate that is shared between the zkVM guest (EVM path) and native verifiers (Solana, CosmWasm). Audit it once; recompiling for a second zkVM backend is a swap, not a rewrite.
- **Fiat–Shamir is where a light-client design lives or dies.** Sample heights must be derived from the freshly-mined tip hash so the prover can't pick favorable samples; this grinding-resistance bound folds directly into the λ security parameter. It is the one place a subtle, silent error would be catastrophic — flagged as an open question for someone who has worked on RFC 45 to sign off.
- **Honest scoping beats a bigger claim.** The asset trust taxonomy correction narrowed the marketing claim but made it true: "the SDK removes the bridge trust, not the custody trust." Writing that distinction into the reference contracts' surfaced metadata is part of the design, not an afterthought.
- **Review before benchmarking.** The Chiral project already has working Eaglesong proving from its CKB→Cardano leg, so the first Phase 0 action became "walk through their accumulator design and Eaglesong proving before committing original benchmarking" rather than reimplementing shared infrastructure.

---

## Plan for next week

- Open the Phase 0 de-risk conversation with T_Silva (Chiral): shared CKB-side Eaglesong proving infrastructure vs. per-target verifiers, and the chain-accumulator proving-cost trade they flagged.
- Scope the Eaglesong-in-SP1 spike (cycles per header, proving time for a 64-header update) against the written kill criteria.
- Draft the cell-liveness question into its own note — inclusion proves a cell was *created*, not that it is currently *unspent*; decide whether to wait on an external live-set commitment from ongoing ZK-on-CKB research before building anything parallel.
