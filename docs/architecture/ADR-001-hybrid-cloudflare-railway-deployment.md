---
status: Proposed
date: 2026-05-31
deciders:
  - aaronsb
  - claude
related:
  - "knowledge-graph-system ADR-055 — CDN & serverless deployment model"
  - "knowledge-graph-system docs/railway-postgres-feasibility.md"
  - "praecipio-intelligence-gateway — deployment pipeline reference"
---

# ADR-001: Hybrid Cloudflare + Railway Deployment Topology for Kappa Graph

## Overview

`kg-cloud` is a thin deployment repository. It carries no application code of its
own; it pulls in Kappa Graph (`knowledge-graph-system`) as a git submodule and
ships it to the cloud with a reproducible, environment-promoted pipeline. This ADR
records *where* each part of Kappa Graph runs and *why* the answer is two providers
rather than one.

## Context

Kappa Graph is four moving parts:

- **viz-app** — a React/TypeScript single-page app.
- **API** — a Python FastAPI service.
- **Database** — Postgres 17 + Apache AGE + a custom Rust `graph_accel` extension.
- **Object storage** — Garage (S3-compatible).

Two prior investigations constrain the design:

1. **ADR-055 (kg)** introduced runtime configuration via `window.APP_CONFIG`
   (`config.js` generated at deploy time) so that one SPA build can be deployed to
   any environment. This is purpose-built for CDN / static hosting.
2. **`railway-postgres-feasibility.md` (kg)** concluded that Railway is a viable
   host for the database, because the stack's only real host requirement is "run my
   Dockerfile, attach a persistent volume at `/var/lib/postgresql/data`, expose
   5432." Railway provides exactly that.

The hard constraint: the `graph_accel` extension is ABI-locked to the `apache/age`
image, loads the entire graph into memory, and requires a persistent volume.
**Cloudflare has no persistent-volume database primitive.** Workers are V8 isolates
(no native Postgres, ~128 MB, no disk). Cloudflare Containers are ephemeral and
scale-to-zero — appropriate for disposable sandboxed compute (as in the praecipio
reference, where a Container backs an ephemeral `Sandbox` Durable Object), but the
wrong shape for a stateful database.

Therefore "deploy all of Kappa Graph on Cloudflare" is not achievable as stated.

## Decision

Adopt a **hybrid topology**: Cloudflare hosts the stateless edge; the stateful core
stays on a volume-capable host (Railway).

**On Cloudflare — owned by this repo:**

1. **Pages** serves the viz-app. Build once from the submodule; inject a per-
   environment `config.js` at deploy time (ADR-055). Same artifact to every env.
2. **R2** replaces Garage for object storage (S3-compatible, zero egress).
3. **Worker gateway (optional, phased)** provides an edge API surface in front of
   the Railway API: request proxy/routing, R2-backed asset serving, an origin trust
   token, and a seam for the future shard-router that ADR-055 §7 sketches as a
   Cloudflare Worker.

**Off Cloudflare — owned by `knowledge-graph-system`:**

4. The FastAPI API and the AGE / `graph_accel` Postgres run on Railway with a
   persistent volume and a backup target (volume snapshot + periodic `pg_dump`).

**Mechanics:**

- Kappa Graph is included as a submodule (`vendor/kg`) **pinned to the `release`
  branch / tags** for reproducible deploys; `make bump-kg` advances it deliberately.
- Environments mirror the praecipio reference: `stage` and `production` branches map
  to GitHub Environments and Cloudflare environments; `production` is gated by a
  required-reviewer rule. Every deploy routes through `make`, so the local loop and
  CI share one code path. Secrets live in GitHub Environment Secrets and are synced
  to Cloudflare at deploy time — never committed.

## Consequences

**Benefits**

- Build-once / deploy-many for the SPA (ADR-055), served from Cloudflare's edge.
- Zero-egress object storage on R2.
- Reproducible deploys: the submodule pin makes "which kg shipped" exact.
- A clean architectural seam (the Worker) for the future shard-router.
- Pipeline parity with a known-good in-house reference (praecipio).

**Trade-offs / costs**

- Two providers to operate and pay for (Cloudflare + Railway).
- Cross-origin traffic between edge and core: CORS configuration plus an origin
  trust token between the Worker/Pages and the Railway API.
- The kg **Operator** mounts the Docker socket for container lifecycle control; that
  pattern translates to neither managed host and is out of scope here.
- R2 <-> Garage is a discrete migration task, not a config flip.
- "Full backend on Cloudflare" is explicitly rejected (see Alternatives).

## Alternatives considered

- **Max-CF via Containers** — run the stateless FastAPI API on Cloudflare
  Containers; the database still needs an external volume host. More Cloudflare
  surface and complexity for no gain on the data layer. Rejected for phase 1;
  revisitable if the API benefits from edge co-location.
- **Cloudflare frontend only** — Pages + R2 and nothing else; all backend stays in
  kg's own deploy. A viable minimal cut, folded in here as Phase 1 rather than the
  end state.
- **Full backend on Cloudflare** — infeasible: no persistent-volume database
  primitive for AGE + `graph_accel`.

## Rollout

| Phase | Scope |
|---|---|
| 1 | Repo + submodule + viz-app -> Pages, `config.js` -> Railway API. Smallest shippable. |
| 2 | R2 bucket; migrate object storage off Garage. |
| 3 | Worker edge gateway (proxy / auth / routing). |
| 4 | Shard-router on the Worker (kg ADR-055 §7), if/when sharding is needed. |
