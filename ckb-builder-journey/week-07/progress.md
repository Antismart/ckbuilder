# Week 7 Progress

**Date:** 4 – 8 May 2026
**Track:** CKBuilder — Community Keeps Building (Nervos)

---

## Overview

Started on the developer dashboard this week. Got two of the ten slices in ADR 0007 done: the frontend scaffold and the backend session-cookie auth that the dashboard sits on. The plan was to have something real running before tackling OAuth, so the rest of the work has somewhere to land instead of just being words in an ADR.

---

## What I worked on

### Dashboard scaffold (slice 1 of ADR 0007)

- Set up `dashboard/` as a new workspace. Vite 6, React 19, TypeScript with strict mode on, Tailwind 4 using the new `@theme` CSS, React Router 7, TanStack Query 5, Vitest, ESLint 9, Prettier 3.
- Pinned pnpm 10.17 in `packageManager` so the toolchain matches across machines.
- Built the basics: landing page, sign-in stub, 404, an `AppShell`, a `NetworkBadge`, and a `useNetwork()` hook that persists to localStorage. Tests for the hook.
- Added `Dockerfile.dev` and a `dashboard` profile in docker-compose so the frontend comes up with the rest of the stack.
- Commit `9719fc8`.

### Backend session auth (slice 2 of ADR 0007)

- Two new tables, `users` and `sessions`, in migration `20260506000001_users_sessions`.
- Repositories and models for both in `crates/db/src/{users,sessions}.rs`. Kept them small — next week's OAuth work just adds creation paths, no schema changes needed.
- Session-cookie middleware in `crates/api/src/session.rs`. Signed cookie, HttpOnly, Secure, SameSite=Lax. The cookie is just an opaque ID; the actual session lives in the database.
- `/admin/me` route group behind the middleware so the dashboard can ask "am I signed in" before OAuth is in.
- E2E tests in `crates/api/tests/api_e2e.rs` for the cookie round-trip and the unauth case.
- Commit `55230ba`.

### Cleanup

- `cbda9ef` — small formatting fix on the `script_registry::lookup` chain in the GraphQL renderer. Saw it while I was in there, three lines, not worth wrapping into a feature commit.

---

## Key learnings

- Standing up the frontend before writing any auth code was a good call. It meant `/admin/me` was designed against an actual caller instead of an imagined one, and the endpoint ended up smaller because the dashboard only needed a yes/no on sign-in.
- `SameSite=Lax` + HttpOnly is the right default here. Same-origin dashboard, no JS-readable cookie, so the OAuth work next week doesn't have to bolt on CSRF protection later.
- The pnpm pin is the kind of thing that's a 30-second fix on day one and a half-day mess if you skip it. Glad I did it now.
- Slicing the dashboard into ten small pieces (per ADR 0007) is already paying off. Two reviewable commits this week instead of one giant "dashboard MVP" PR.

---

## Blockers

- Nothing in the way.  

---

## Plan for next week

 
- Slice 3: GitHub OAuth. Callback handler, code exchange, session issuance against the new tables. The middleware from this week is what it plugs into.
- Slice 4: hook the dashboard sign-in stub up to the real OAuth flow and make `/admin/me` the gate for the authenticated routes.
- Start slice 5 if there's time: API key management UI (list, issue, revoke) on top of last week's `api_keys` schema.
