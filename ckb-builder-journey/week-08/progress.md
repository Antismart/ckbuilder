# Week 8 Progress

**Date:** 11 May - 16 May 2026
**Track:** CKBuilder — Community Keeps Building (Nervos)
**Project:** Cellora — managed indexing and query layer for CKB

---

## Overview

This week shifted from the Rust backend to the frontend — the Cellora dashboard is now a separate Nuxt 3 application with a full landing page, authenticated dashboard shell, API key management UI, and a live network switcher. The backend and frontend are now two independently deployable services.

---

## What shipped

### cellora-nuxt — Nuxt 3 dashboard (new repo)

**Repo:** https://github.com/Antismart/cellora-nuxt

**Tech stack:** Nuxt 3, Vue 3 `<script setup>` + Composition API, TypeScript. No external UI library — all components are in-repo with a custom design token system in CSS.

### Landing page
- Hero section, Features section, and Pricing section built as separate components under `components/landing/`
- Three SVG diagrams under `components/landing/diagrams/` illustrating the core value props (no-node querying, multi-tier access, live data layer)
- Clean design tokens and utility classes in `assets/css/main.css`

### Authentication and routing
- `middleware/auth.ts` gates all `/app/*` routes — unauthenticated users are redirected to sign-in
- Sign-in page at `pages/sign-in.vue`
- SSR-safe auth state via `useAuth` composable

### Dashboard shell
- `layouts/dashboard.vue` — sidebar and topbar shell wrapping all authenticated routes
- `NetworkSwitcher` component for toggling between mainnet and testnet
- `UserMenu` component for account actions
- `useLiveTip` composable polling the indexer tip in real time
- `useNetwork` composable managing network selection state across the app

### UI component library
Built from scratch, no external dependency:
- `Button`, `Card`, `Badge`, `Modal`, `Tabs`
- `Icon.vue` — name-based icon registry for consistent iconography across the app
- Charts and API key management modals under `components/dashboard/`

### Utilities
- Formatters for CKB values, timestamps, and hex addresses
- Mock data layer for local development without a live backend
- Syntax highlighter utility for API docs and code examples

---

## Architecture note

The decision to use Nuxt rather than the React + Vite stack originally planned in the roadmap was deliberate. Nuxt's SSR-first model is a better fit for the landing page (SEO matters for developer tools discovery) while the `app/*` dashboard routes run as a SPA. The composables pattern maps cleanly onto the real-time data requirements (`useLiveTip`, `useNetwork`) without needing a state management library.

---

 

## Key learnings
 - SSR-safe composables require more discipline than client-only state — `useAuth` had to be written to handle the hydration boundary correctly so it doesn't flash unauthenticated state on page load

---

## Plan for next week
- Connect the dashboard to the live Rust API (replace mock data with real calls)
