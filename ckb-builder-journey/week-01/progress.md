# Week 1 Progress

**Date:** 24 - 28 March 2026
**Track:** CKBuilder — Community Keeps Building (Nervos)

---

## What I worked on

### CKB Academy
- Completed modules 1 through 5 out of 8 this week
- Module 1-2: Got a solid intro to CKB's architecture and the **Cell model** — how it manages state differently from account-based chains like Ethereum
- Module 3-4: Went through the L1 developer training module (Lumos-based) — framework is outdated but the underlying concepts around transaction structure and scripting were still very useful
- Module 5: Currently here — continuing to build on the foundations from the earlier modules
- Set up the local dev environment and got the Rust boilerplate running using the `ckb-script-templates` workspace
 

### Hackathon
- Attempted to build and submit a project for the **Claw & Order: CKB AI Agent Hackathon**
- Missed the submission deadline (25th March 12:00 UTC) due to not seeing the announcement in time
- Still a useful exercise — gave me a practical reason to start building on CKB early

---

## Key learnings
- CKB's Cell model is closer to Bitcoin's UTXO model than Ethereum's account model
- Scripts in CKB are lock scripts and type scripts attached to cells — very different mental model from Solidity
- The `riscv64imac-unknown-none-elf` Rust target is required for compiling CKB scripts

---

## Blockers

### ckb-debugger install failure
- Running `cargo install --git https://github.com/nervosnetwork/ckb-standalone-debugger ckb-debugger` was failing with 37 compile errors in `ckb-jsonrpc-types v1.0.0`
- Root cause: Cargo was resolving `schemars v1.x` which introduced breaking changes — `H256` implements `schemars::JsonSchema` but not the version expected by `ckb_schemars-0.8.22`
- Tried `--locked` flag but it had no effect because the first install attempt without `--locked` had already cached a broken dependency resolution in Cargo's git store
- **Fix:** Cloned the repo locally and ran `cargo install --path . --locked` from inside it — this ensured the pinned `Cargo.lock` in the repo was respected and the correct dependency versions were used

---