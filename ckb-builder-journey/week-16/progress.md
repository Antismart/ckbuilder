# Week 16 Progress

**Date:** 28 July 2026
**Track:** CKBuilder — Community Keeps Building (Nervos)
**Project:** Cellora — managed indexing and query layer for CKB

---

## Overview

A design-consistency pass on the Cellora frontend (the Nuxt dashboard + landing site), notable less for the CSS than for how it was done. The task was a set of flat-UI rules — no gradients, no glow effects, no decorated initial boxes — and rather than sweep them by hand I encoded each rule as a reusable agent "skill" and ran a multi-agent workflow to apply them, with a verifier to prove the result.

---

## What shipped

### The visual cleanup

Removed decoration across the dashboard and landing surfaces (net ~35 lines *deleted*):

- **Gradients → flat solids:** the user-menu avatar tile, the SVG chart area-fills (now flat low-opacity fills), the global dot-grid / grid-line / skeleton-shimmer / mask-fade patterns in `main.css`, the `Select` dropdown caret (now a flat SVG chevron), and the CTA/overview mask fades.
- **Glow effects removed:** the `.hero-glow` radial halo and the Pricing featured-card brand-tinted ring.
- **Decorated initial box removed:** the user-menu avatar is now a flat surface tile with a plain border instead of a green→blue gradient.

The one thing deliberately *kept*: neutral black elevation shadows on cards, modals, and popovers, plus the frosted `backdrop-blur` nav. That distinction — depth is not glow — was the whole game.

### The tooling: rules as reusable skills

Wrote three skill files (`no-gradients`, `no-glow-effects`, `no-decorated-initials`) that codify each rule as a durable artifact: the violation patterns to grep for, the fix recipe, and — critically — the *boundaries* (what to keep). The `no-glow-effects` skill spells out glow-vs-elevation explicitly so an agent removes a colored halo but never a functional depth shadow. They live in the frontend repo for reuse on future UI work.

### The workflow

Four specialist agents, each owning a **disjoint** slice of files (landing / dashboard / ui-primitives / pages+global-CSS) so they could edit in parallel without colliding, each loading all three skills and applying them. A final verifier grepped the whole source for residual gradients/glows/decorated-initials, confirmed the elevation shadows were preserved (not over-stripped), and sanity-checked that every touched `.vue` file still had balanced blocks. Zero residual violations, no depth regressions.

---

## Key learnings

- **A design rule is safely delegable only once it's written down with its exceptions.** The valuable line in the skills wasn't "remove glows" — it was "keep neutral elevation shadows and backdrop-blur glass." Without that boundary a keen agent flattens the entire UI; with it, the pass is clean. Encoding the *keep* list mattered more than the *remove* list.
- **Disjoint file ownership is the reliable way to parallelize edits.** Same lesson as the backend fix workflow the week before, reused deliberately: partition by file, not by task, and concurrent agents never conflict.
- **A grep-based verifier turns "looks done" into "is done."** Having a separate agent re-scan for residual patterns and check that nothing protected was removed caught the difference between a plausible summary and a verified result — worth the extra step every time.

---

## Plan for next week

- Still the same gating item on the backend: clear the GitHub Actions billing block so CI finally runs the integration suite end to end.
- Close the remaining Cellora backend gaps from the production-readiness review — the rate-limiter fail-open decision, the webhook retry queue, and the deployment runbook.
- Eyeball the deployed frontend once Vercel rebuilds to confirm the flat-UI pass reads well in the live dashboard.
