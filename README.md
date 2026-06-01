# kg-cloud

![License](https://img.shields.io/github/license/aaronsb/kg-cloud)

Cloud deployment repository for **[Kappa Graph — κ(G)](https://github.com/aaronsb/knowledge-graph-system)**.

This repo does not contain the application. It contains the *deployment* of it.
Kappa Graph is pulled in as a git submodule (`vendor/kg`, pinned to its `release`
branch) and shipped to a hybrid cloud topology with a reproducible, environment-
promoted pipeline.

## Honest topology: it's two clouds, not one

The name says "cloud," not "cloudflare," on purpose. Kappa Graph doesn't fit on a
single serverless provider, and pretending otherwise would be dishonest. The data
layer is genuinely stateful — Apache AGE Postgres with a custom Rust `graph_accel`
extension that loads the whole graph into RAM and is ABI-locked to the `apache/age`
image. Cloudflare has no persistent-volume database primitive, so the database
lives where it can: **Railway** (see the kg feasibility study).

```mermaid
%%{init: {"flowchart": {"curve": "basis"}, "themeVariables": {"fontSize": "14px"}}}%%
flowchart TB
    Browser(["🌐 Browser"])

    subgraph CF["☁️ Cloudflare — edge (this repo)"]
        direction TB
        Pages["<b>Pages</b> · kg-viz-&lt;env&gt;<br/>viz-app SPA, build-once<br/>+ /config.js per-env (ADR-055)"]
        Worker["<b>Worker</b> · kg-gateway-&lt;env&gt;<br/>proxy / route · R2 binding<br/>Hyperdrive (if PG direct)"]
        R2[("<b>R2</b> · kg-assets-&lt;env&gt;<br/>object storage<br/>(replaces Garage)")]
    end

    subgraph RW["🚂 Railway — core (knowledge-graph-system)"]
        direction TB
        API["<b>API</b> · FastAPI"]
        DB[("<b>Postgres 17</b> + Apache AGE + graph_accel<br/>persistent volume · in-RAM graph cache")]
    end

    Browser -->|"static"| Pages
    Browser -->|"/api/*"| Worker
    Worker --> R2
    Worker -->|"HTTPS · origin trust token"| API
    API --> DB

    %% Dual-theme palette: explicit text colors per node so legibility
    %% never depends on the viewer's light/dark background.
    style CF fill:#f6821f1a,stroke:#d97706,stroke-width:2px,color:#d97706
    style RW fill:#7c3aed1a,stroke:#8b5cf6,stroke-width:2px,color:#8b5cf6
    style Browser fill:#475569,stroke:#94a3b8,stroke-width:2px,color:#ffffff
    style Pages fill:#f6821f,stroke:#9a3412,stroke-width:1px,color:#1a1a1a
    style Worker fill:#f6821f,stroke:#9a3412,stroke-width:1px,color:#1a1a1a
    style R2 fill:#fbbf24,stroke:#92400e,stroke-width:1px,color:#1a1a1a
    style API fill:#7c3aed,stroke:#4c1d95,stroke-width:1px,color:#ffffff
    style DB fill:#6d28d9,stroke:#4c1d95,stroke-width:1px,color:#ffffff
```

| Lives on Cloudflare | Lives on Railway |
|---|---|
| viz-app (Pages) | FastAPI API |
| Object storage (R2, replaces Garage) | Postgres 17 + Apache AGE + `graph_accel` |
| Edge API gateway (Worker, optional) | Persistent volume + backups |

The decision and its alternatives are recorded in
[ADR-001](docs/architecture/ADR-001-hybrid-cloudflare-railway-deployment.md).

## Layout

```
kg-cloud/
├── vendor/kg/              # submodule -> knowledge-graph-system @ release
├── apps/gateway/           # CF Worker edge gateway (phase 3)
├── config/                 # window.APP_CONFIG templates, per environment
├── scripts/                # build-web, gen-config, bump-kg
├── wrangler.toml           # multi-env: dev / stage / production
├── Makefile                # install / build / deploy / secrets-sync
├── .github/workflows/      # deploy.yml -- branch -> environment
└── docs/architecture/      # ADRs
```

## Environments

Mirrors the praecipio deployment reference: branch -> GitHub Environment ->
Cloudflare environment, all routed through `make` so local and CI share one path.

| Branch | GH Environment | Cloudflare env | Notes |
|---|---|---|---|
| `main` | `production` | `production` | gated by required-reviewer |
| `stage` | `stage` | `stage` | staging |
| (local) | — | `dev` | `make dev` |

Secrets (`CLOUDFLARE_API_TOKEN`, `CLOUDFLARE_ACCOUNT_ID`, API origin token, …)
live in GitHub Environment Secrets and are synced to Cloudflare at deploy time.
None are committed to this repo.

## Updating Kappa Graph

The submodule is pinned to a specific kg `release` commit for reproducible
deploys. To ship a newer kg:

```bash
make bump-kg        # advance vendor/kg to the latest kg release tag
git commit vendor/kg -m "chore(kg): bump to <tag>"
```

## Status

Phase 1 (foundation + viz-app -> Pages). Pipeline lands in follow-up PRs.

## License

Apache-2.0, matching [knowledge-graph-system](https://github.com/aaronsb/knowledge-graph-system).
