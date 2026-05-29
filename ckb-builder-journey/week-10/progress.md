# Week 10 Progress

**Date:** 24 May - 29 May 2026
**Track:** CKBuilder — Community Keeps Building (Nervos)
**Project:** Cellora — managed indexing and query layer for CKB

---

## Overview

This week, we shifted our focus toward production readiness and observability. We replaced the mock data in the developer dashboard with real API consumption metrics, implemented a non-blocking request logging system via Redis streams, and successfully deployed the entire backend stack to a VPS while hosting the frontend on Vercel.

---

## What shipped

### cellora — Production Deployment and Analytics

We finalized the backend metrics endpoints and deployed the API to our VPS.

- **Non-blocking Request Logging:** To prevent database writes from slowing down API responses, we implemented an async worker (`metrics_worker.rs`). The API now pushes lightweight request summaries (method, path, status, latency) to a Redis stream, and a background worker batches these writes into the `api_request_logs` PostgreSQL table.
- **Dashboard Metrics Endpoints:** Added new routes (`/admin/metrics/summary`, `/admin/metrics/endpoints`, `/admin/metrics/rate-limits`) to serve aggregate usage data, error rates, p95 latencies, and rate-limit hits to the dashboard in real-time.
- **VPS Deployment:** Deployed the full stack (Axum API, indexer, PostgreSQL, Redis, and Nervos CKB node) to a VPS. We moved from local development to a structured `docker-compose.prod.yml` setup, handling port bindings, health checks, and restart policies for all services.

---

### cellora-nuxt — Vercel Deployment and Real Data Integration

We moved the frontend off localhost and connected it to the live backend.

- **Proxy Routing Fixes:** When deploying to Vercel, the Nitro `devProxy` was dropping query parameters on the OAuth callback (`?code=...&state=...`). We replaced this with an explicit `proxyRequest` implementation using `h3` that faithfully forwards all headers, cookies, and query strings from the edge to the VPS backend.
- **Real Metrics Consumption:** Ripped out the hardcoded mock data on the Overview and Usage pages. The dashboard now pulls live metrics from the backend, rendering real charts for API requests, latency distributions, and active endpoints.
- **Cleanup and Polish:** Stripped out unused mock files, cleaned up the navigation structure, and rewrote the main Cellora README to focus strictly on the architecture and product offering.

---

## Key learnings

- **Proxying OAuth flows across domains is tricky:** Vercel handles the SSL termination and serves the frontend on HTTPS, but the backend lives on a separate VPS. Carefully forwarding cookies, query parameters, and ensuring the `redirect_uri` matches exactly was critical to getting GitHub OAuth to work across this boundary.
- **Redis streams for analytics:** Writing directly to Postgres on every API hit introduces too much latency. Using Redis streams as a buffer for the metrics worker proved to be a simple and effective pattern for decoupling request serving from analytics processing.

---

## Plan for next week

- Monitor the live deployment on the VPS and address any memory or performance bottlenecks.
- Begin work on integrating the custom CKB indexing logic to expose more advanced query capabilities for dApp developers.
- Continue polishing the frontend UI and adding more detailed charting options.
