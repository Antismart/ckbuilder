# Week 9 Progress

**Date:** 18 May - 23 May 2026
**Track:** CKBuilder — Community Keeps Building (Nervos)
**Project:** Cellora — managed indexing and query layer for CKB

---

## Overview

This week, work focused on bridging the newly designed Nuxt 3 dashboard frontend with the Rust backend via a secure, production-ready authentication system. We completed the server-mediated GitHub OAuth flow, implemented database-backed session management using HttpOnly cookies, resolved cross-origin issues between the two services, and successfully wired the frontend to authenticate against the live API.

---

## What shipped

### cellora — Axum API and OAuth backend

We completed Slice 3 and Slice 4 of the developer dashboard roadmap outlined in ADR 0007:

#### GitHub OAuth implementation
- Added `GET /admin/oauth/github/start` to generate a random 32-byte `state` token, persist it to Redis with a 10-minute expiry (mitigating CSRF), and redirect the browser to the GitHub authorize URL.
- Added `GET /admin/oauth/github/callback` to handle GitHub's authorization code redirect. It exchanges the authorization code for an access token, fetches the user's GitHub profile and verified primary email, and upserts a row in the `users` table.
- Added `POST /admin/sign-out` to invalidate the session in the database and instruct the browser to clear the session cookie.

#### Session management with signed cookies
- Added the `sessions` table schema and model to handle active sessions securely.
- Developed a custom Axum session middleware that extracts the `cellora_session` cookie on every `/admin/*` request, hashes it (SHA-256) to look up the session in PostgreSQL, checks for expiration (30-day rolling TTL), resolves the user, and attaches the active session details to the request extensions.
- Configured the session cookie with industry-standard security properties: `HttpOnly`, `SameSite=Lax`, `Path=/`, and conditional `Secure` flags (relaxed only in dev mode).
- Added comprehensive E2E integration tests in `tests/api_e2e.rs` validating cookie authentication round-trips, unauthenticated rejections, and expired session handlings.

#### Configurable CORS and Local Dev Proxy
- Added `tower-http::cors::CorsLayer` configuration to allow allow-listed dashboard origins to hit the backend API while carrying cookies (`Access-Control-Allow-Credentials: true`).
- Configured a local development proxy inside Nuxt 3 to forward `/admin/*` and `/v1/*` requests to the Rust API dev server (`http://localhost:8080`), ensuring same-origin cookie behavior and an effortless local setup.

---

### cellora-nuxt — Nuxt 3 dashboard integration

With the backend OAuth and session layers completed, we moved the Nuxt 3 dashboard prototype off pure local mocks:

- **Live Auth Integration:** Updated the `useAuth` composable (`composables/useAuth.ts`) to hit `GET /admin/me` on startup. If the backend is running and responds with a valid profile, it signs the user in; if the server rejects with a 401, it bounces them to `/sign-in`.
- **Offline / Mock Mode Graceful Degradation:** Built a robust fail-safe mechanism where the dashboard automatically detects if the backend is unreachable (e.g. during offline frontend development) and flips into a local mock mode (`usingMock = true`) so visual iteration isn't blocked.
- **Wired Sign-In and Sign-Out Flows:** Wired the "Continue with GitHub" button to perform a full browser redirect to the backend's `/admin/oauth/github/start` path in live mode, and wired the user avatar menu's "Sign Out" option to post to `/admin/sign-out` before redirecting to the landing page.

---

## Key learnings

- **Separating surfaces pays off:** Keeping programmatic API keys (`/v1/*`) and human dashboard sessions (`/admin/*`) completely isolated—using separate tables, authentication logic, and credentials—made it easy to reason about security boundaries and rate-limiting limits.
- **Fail-safe mock modes are a developer's best friend:** Adding `usingMock` to the Nuxt frontend allowed team members to keep building UI features without constantly needing a running local backend, CKB dev node, and real GitHub OAuth credentials.
- **Signed cookies over bearer tokens for dashboards:** Gating the dashboard behind an opaque, signed `cellora_session` cookie ensures that the browser never holds or exposes raw bearer tokens in local storage, significantly shrinking the XSS attack vector.

---

## Plan for next week

- **Slice 5: API Key Management (User Scoping)**
  - Migrate `api_keys` to include a `user_id` foreign key referencing the `users` table and a stable surrogate UUID `id` to allow rotation (which changes the prefix/secret but retains key identity).
  - Implement `/admin/keys` CRUD endpoints on the backend Axum API (listing, creation, rotation, and revocation scoped to the logged-in user).
  - Connect the dashboard keys page (`pages/app/keys.vue`) to the real backend endpoints, allowing users to create, rotate, and revoke their API keys in real time.
