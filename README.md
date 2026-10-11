# 9router-deploy

Deployment stack for [9router](https://github.com/lobehub/lobe-chat), an LLM gateway and chat interface. This repository provides Docker Compose configuration for self-hosting 9router with Cloudflare Tunnel ingress and automated CI/CD deploys to a Tencent Cloud VPS.

## Architecture

```mermaid
flowchart LR
    subgraph Internet
        User([User])
    end
    subgraph Cloudflare
        CF[Cloudflare Edge]
        T[Tunnel]
    end
    subgraph Docker["Docker Compose Stack"]
        cloudflared[cloudflared]
        caddy[Caddy]
        api[9router-api :20127]
        dashboard[9router :20128]
        headroom[headroom :8787]
    end
    subgraph Storage
        DB[(SQLite)]
    end

    User --> CF --> T --> cloudflared --> caddy
    caddy --> api
    caddy --> dashboard
    dashboard --> headroom
    api --> DB
    dashboard --> DB
```

**Traffic flow:** User → Cloudflare Edge → Cloudflare Tunnel → `cloudflared` → `caddy` (reverse proxy) → `9router-api` / `9router`.

TLS is terminated at the Cloudflare edge; Caddy inside the stack receives plain HTTP from the tunnel and routes by hostname.

## Infrastructure

### Deployment host layout

The stack deploys to `/opt/9router` on the VPS:

```
/opt/9router
├── docker-compose.yml          # Production stack
├── proxy/Caddyfile             # Reverse proxy config
├── src/9router/Dockerfile      # Dashboard image build (manual-rebuild fallback)
├── src/9router-api/Dockerfile  # API image build (manual-rebuild fallback)
├── env/                        # Per-service env files (never committed)
├── cloudflared/                # Tunnel credentials JSON + config
└── data/                       # Shared SQLite data + home volumes
```

> Production images are built on GitHub runners and pulled from GHCR; the Dockerfiles on the host are unused for the production path and retained only for manual rebuilds.

### Services

| Service | Port | Description |
|---------|------|-------------|
| `9router` | 20128 | Next.js dashboard UI (profile-gated, optional) |
| `9router-api` | 20127 | Express LLM proxy API |
| `headroom` | 8787 | Token compression/shaping proxy |
| `caddy` | 80 | Reverse proxy (plain HTTP, TLS at Cloudflare edge) |
| `cloudflared` | — | Cloudflare Tunnel connector |

All services share the `9router-net` bridge network. The VPS exposes no public app ports — ingress is exclusively via the Cloudflare Tunnel.

### Routing (Caddy)

`cloudflared` forwards tunneled requests to `caddy:80`; Caddy routes by hostname:

- `dashboard.example.com` → `9router:20128`
- `api.example.com` → `9router-api:20127`

### Data persistence

`9router` and `9router-api` share storage via bind mounts:

- `./data/9router:/app/data` — shared SQLite database
- `./data-home/9router-home:/app/data-home` — shared home volume

## Setup

### Prerequisites

- Docker & Docker Compose v2+
- Cloudflare account with Zero Trust (free tier works)
- A domain managed by Cloudflare
- (For CI/CD) A VPS reachable over SSH, e.g. Tencent Cloud

### 1. Clone and prepare env files

```bash
git clone https://github.com/your-org/9router-deploy.git
cd 9router-deploy

mkdir -p env data/9router data-home/9router-home
cp env/9router.env.example env/9router.env
cp env/9router-api.env.example env/9router-api.env
# Create env/caddy.env and env/cloudflared.env manually (see below)
```

Set `JWT_SECRET` to the same value in both `env/9router.env` and `env/9router-api.env`.

### 2. Run locally (no tunnel)

```bash
docker compose -f docker-compose.yml -f docker-compose.local.yml --profile dashboard up -d
```

Access locally:
- API: `http://localhost:20127/api/health`
- Dashboard: `http://localhost:20128`

The local override keeps `caddy` and `cloudflared` stopped; only `9router`, `9router-api`, and `headroom` run.

### 3. Cloudflare Tunnel and DNS guide

1. In the Cloudflare dashboard, go to **Zero Trust → Networks → Tunnels** and **Create a tunnel**. Pick the **Cloudflared** connector type.
2. Name the tunnel (e.g. `9router-prod`) and copy its token into `env/cloudflared.env`:

```bash
TUNNEL_TOKEN=<paste-cloudflare-tunnel-token-here>
```

3. Configure public hostnames pointing to the Caddy service:

| Public hostname | Service |
|-----------------|---------|
| `dashboard.example.com` | `http://caddy:80` |
| `api.example.com` | `http://caddy:80` |

4. Match the hostnames in `env/caddy.env`:

```bash
SITE_DASHBOARD=dashboard.example.com
SITE_API=api.example.com
```

5. Cloudflare creates the DNS CNAME records automatically (`<tunnel-id>.cfargotunnel.com`). To add them manually:

```bash
dashboard.example.com  CNAME  <tunnel-id>.cfargotunnel.com
api.example.com        CNAME  <tunnel-id>.cfargotunnel.com
```

6. Verify the tunnel status is **Healthy** in the Zero Trust dashboard, then deploy.

> Never commit tunnel credentials — `env/*.env` and `cloudflared/*.json` are gitignored, and the credentials JSON should be mode `600` on the server.

## Deployment

### Manual (production, with tunnel)

Production images are prebuilt on GitHub runners and published to GHCR:

```bash
docker compose --profile dashboard pull 9router 9router-api headroom
docker compose --profile dashboard up -d
docker compose ps
```

For local development from source, use the local override (builds instead of pulling):

```bash
docker compose -f docker-compose.yml -f docker-compose.local.yml --profile dashboard up -d
```

### CI/CD with GitHub Actions

The workflow `.github/workflows/deploy.yml` deploys on:
- push to `master`
- manual `workflow_dispatch` (with optional `ninerouter_branch` / `api_branch` inputs)
- `repository_dispatch` of type `deploy`

**Required repository secrets** (GitHub → Settings → Secrets and variables → Actions):

| Secret | Description |
|--------|-------------|
| `TENCENT_HOST` | Deployment server hostname/IP |
| `TENCENT_USER` | SSH username |
| `TENCENT_SSH_KEY` | SSH private key for deployment |

Images are built on GitHub-hosted runners and pushed to GitHub Container Registry (`ghcr.io/vianhanif/9router-deploy/*`), not on the VPS. The packages are public, so the VPS pulls anonymously and needs no registry credentials. The workflow job carries an explicit `permissions:` block (`contents: read`, `packages: write`, `actions: write`) because the repository's default workflow-token permission is read-only.

**Pipeline steps:**

1. **Resolve source SHAs** — resolves the `9router` / `9router-api` refs to commit SHAs (`repository_dispatch` carries no inputs and falls back to `master`).
2. **Sync config** — copies `docker-compose.yml`, `proxy/Caddyfile`, and both Dockerfiles to `/opt/9router` via SCP (Dockerfiles are kept on the host for manual rebuild fallback).
3. **Build & push** — `docker/build-push-action` builds `9router` and a single image tagged for both `9router-api` and `headroom`, pushing `:latest` plus a `:sha-<sha>` audit tag to GHCR with GitHub Actions build cache.
4. **Smoke test** — runs each new image on scratch ports (`20328`/`20327`) and verifies `/api/health` plus the `/v1/responses` route. Uses the SHA-tagged refs so the implicit pull does not disturb the running `:latest` that `:prev` is about to be captured from. The deploy aborts if any check fails.
5. **Switch** — tags the current `:latest` images as `:prev`, pulls the new images, then recreates the stack (`--force-recreate`).
6. **Prune** — removes `:prev` images, stale `:sha-*` tags from earlier deploys, dangling images, and build cache (only after a successful deploy).

**Rollback:** if any step after the smoke test fails, the `rollback` job restores `:prev` → `:latest` and recreates the stack. This works purely from images already present on the VPS, so it needs neither registry access nor a rebuild.

### Preview deployments (non-master branches)

`.github/workflows/preview.yml` spins up an **isolated** stack so a branch can be tested on its own hostnames without touching production.

| | Production | Preview |
|---|---|---|
| Compose project | `9router` | `9router-preview` |
| Compose file | `docker-compose.yml` | `docker-compose.preview.yml` |
| Caddyfile | `proxy/Caddyfile` | `proxy/Caddyfile.preview` |
| Containers | `9router`, `9router-api`, `headroom`, `caddy` | `9router-test`, `9router-api-test`, `headroom-test`, `caddy-test` |
| Data | `./data/9router` | `./data-preview/9router` |
| Hostnames | `9router.vianhanif.link`, `9router-dashboard.vianhanif.link` | `9router-test.vianhanif.link`, `9router-dashboard-test.vianhanif.link` |

**Triggers:** `workflow_dispatch` or `repository_dispatch` of type `preview`.

**Inputs:**

| Input | Values | Notes |
|---|---|---|
| `branch` | branch name or 40-char SHA | source for **both** 9router and 9router-api builds |
| `type` | `select` \| `dashboard` \| `api` \| `both` | `select` + non-master is a hard failure; `select` + `master` resolves to `both` |
| `action` | `deploy` \| `teardown` | teardown runs `docker compose ... down -v` |

**Data seeding:** on the first deploy, `./data/9router` is copied to `./data-preview/9router` with `rsync`. The preview then owns its copy and never writes back to production data.

**Networking:** `caddy-test` joins the production Docker network (`9router_9router-net`) as an external network so the already-running `cloudflared` can reach it, while the preview containers themselves stay on their own `9router-preview-net`.

**Prerequisite — Cloudflare ingress (manual, one-time):** the tunnel is token-managed, so ingress rules live in the Cloudflare Zero Trust dashboard, not in this repo. Add two public hostnames pointing at the same origin service:

```
9router-test.vianhanif.link            ->  http://caddy-test:80
9router-dashboard-test.vianhanif.link  ->  http://caddy-test:80
```

**No rollback:** a broken preview is torn down, not rolled back. The workflow has no `:prev` snapshot logic.

**Concurrency:** the preview lock (`preview-9router-deploy`) is separate from the production lock (`deploy-9router`), so the two pipelines never block each other.

## Environment Variables

### 9router.env

| Variable | Description | Example |
|----------|-------------|--------|
| `BASE_URL` | Public URL of the dashboard | `https://dashboard.example.com` |
| `CLOUD_URL` | Cloud/sync URL (usually same as BASE_URL) | `https://dashboard.example.com` |
| `JWT_SECRET` | Secret for JWT signing (must match 9router-api) | `<random-string>` |
| `DATA_DIR` | Data directory path inside container | `/app/data` |
| `PORT` | HTTP port | `20128` |
| `NODE_ENV` | Environment mode | `production` |

### 9router-api.env

| Variable | Description | Example |
|----------|-------------|--------|
| `PORT` | HTTP port | `20127` |
| `NODE_ENV` | Environment mode | `production` |
| `NINEROUTER_HOME` | Path to 9router source | `/app/9router` |
| `DATA_DIR` | Data directory path inside container | `/app/data` |
| `BASE_URL` | Public URL | `https://dashboard.example.com` |
| `CLOUD_URL` | Cloud/sync URL | `https://dashboard.example.com` |
| `JWT_SECRET` | Secret for JWT signing (must match 9router) | `<random-string>` |

### caddy.env

| Variable | Description | Example |
|----------|-------------|--------|
| `SITE_DASHBOARD` | Hostname for dashboard | `dashboard.example.com` |
| `SITE_API` | Hostname for API | `api.example.com` |

### cloudflared.env

| Variable | Description |
|----------|-------------|
| `TUNNEL_TOKEN` | Cloudflare Tunnel token from Zero Trust dashboard |

## Repository Structure

```
├── docker-compose.yml          # Production stack
├── docker-compose.local.yml    # Local dev overrides
├── .env.example                # Environment template
├── env/                        # Per-service env files (gitignored)
│   ├── 9router.env.example
│   └── 9router-api.env.example
├── proxy/
│   └── Caddyfile               # Reverse proxy config
├── cloudflared/
│   └── config.yml.example      # Tunnel config template
├── src/
│   ├── 9router/                # Dashboard source
│   └── 9router-api/            # API source
└── .github/workflows/
    └── deploy.yml              # CI/CD workflow
```

## Security

- Never commit real secrets: `.env`, `env/*.env`, and `cloudflared/*.json` are gitignored.
- The CI/CD pipeline reads credentials exclusively from GitHub Actions secrets (`TENCENT_HOST`, `TENCENT_USER`, `TENCENT_SSH_KEY`).
- Production images are built on GitHub-hosted runners in a clean environment with no long-lived secrets. GHCR packages are public (free tier, no quota charges), so pulls from the VPS are anonymous with no registry credentials.
- Keep tunnel credentials mode `600` on the server.
- This repository contains no real secrets; all examples use placeholder values.

## License

MIT
