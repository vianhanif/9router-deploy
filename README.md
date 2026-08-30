# 9router-deploy

Deployment stack for the **9router** + **9router-api** services on Tencent Cloud (Tencent Cloud VPS, `/opt/9router`).

This repo mirrors the application source needed to build the stack: the dashboard fork (`vianhanif/9router`) lives under `src/9router/` and the API under `src/9router-api/`. Both are built from source at deploy time; no binary images are pulled for the app services.

## Architecture

| Service | Image / Build | Port | Role |
|---------|---------------|------|------|
| `9router` | built from `src/9router/` (fork `vianhanif/9router`) | 20128 | Next.js monolith + dashboard UI |
| `9router-api` | built from `src/9router-api/Dockerfile` | 20127 | Standalone Express LLM proxy (no UI) |
| `caddy` | `caddy:alpine` | 80/443 | Reverse proxy / TLS ingress |
| `cloudflared` | `cloudflare/cloudflared:latest` | — | Cloudflare Tunnel to `vianhanif.link` |

Public hostnames served (all via Cloudflare Tunnel `9router-tencent` → VPS `cloudflared` → `caddy`):
- `9router-dashboard.vianhanif.link` → dashboard UI (`9router:20128`)
- `9routerapi.vianhanif.link` → API (`9router-api:20127`)
- `9router.vianhanif.link` → API (`9router-api:20127`) — the endpoint used by Warp's custom endpoint

Both `9router` and `9router-api` share the **same SQLite database** via a bind mount (`./data/9router:/app/data`). `9router` runs DB migrations on boot; `9router-api` is read-mostly and `depends_on` a healthy `9router`.

External path: public host → Cloudflare Tunnel → `cloudflared` (port 443) → `caddy` (internal `:80`) → app service.

## Repo layout

```
.
├── docker-compose.yml          # full 4-service stack
├── .env.example                # env template (no real secrets)
├── .gitignore                  # ignores .env, env/*.env, cloudflared/*.json, data/, data-home/, src/9router/{node_modules,.next}
├── env/                        # per-service env (gitignored — contains secrets)
│   ├── 9router.env
│   ├── 9router-api.env
│   ├── caddy.env
│   └── cloudflared.env         # TUNNEL_TOKEN (remotely-managed tunnel)
├── proxy/
│   └── Caddyfile               # reverse-proxy rules (dashboard + api hosts)
├── cloudflared/
│   ├── config.yml              # tunnel ingress (gitignored real file)
│   └── config.yml.example
├── .github/workflows/deploy.yml
└── src/
    ├── 9router/                # fork source for docker compose build 9router (node_modules/.next gitignored)
    │   ├── Dockerfile
    │   ├── custom-server.js
    │   ├── open-sse/ …
    └── 9router-api/            # source for docker compose build 9router-api
        ├── Dockerfile
        └── .dockerignore
```

## Remote layout (`/opt/9router`)

```
/opt/9router/
├── docker-compose.yml
├── env/                        # real env files (gitignored)
│   └── cloudflared.env         # TUNNEL_TOKEN (remotely-managed tunnel)
├── data/9router/               # bind-mount -> /app/data (SQLite DB, secrets, runtime state)
├── data-home/9router-home/     # bind-mount -> /app/data-home
├── src/9router/                # 9router fork source (build context)
├── src/9router-api/            # 9router-api source (build context)
├── proxy/
└── cloudflared/                # config.yml + <tunnel-id>.json (credentials, never committed)
```

## Environment setup

The application has **no `.env` file** — it read configuration from pm2 process env and files in `~/.9router/` on the origin machine. For the container deployment the equivalent values are placed in `env/*.env`:

**`env/9router.env`**
```
BASE_URL=https://9router-dashboard.vianhanif.link
CLOUD_URL=https://9router-dashboard.vianhanif.link
JWT_SECRET=<jwt secret from ~/.9router/jwt-secret>
DATA_DIR=/app/data
PORT=20128
NODE_ENV=production
```

**`env/9router-api.env`**
```
PORT=20127
NODE_ENV=production
NINEROUTER_HOME=/app/9router
DATA_DIR=/app/data
BASE_URL=https://9router-dashboard.vianhanif.link
CLOUD_URL=https://9router-dashboard.vianhanif.link
JWT_SECRET=<same jwt secret>
```

**`env/caddy.env`**
```
SITE_DASHBOARD=9router-dashboard.vianhanif.link
SITE_API=9routerapi.vianhanif.link
```

Notes:
- JWT_SECRET is provisioned through the deployment environment.
- `NEXT_PUBLIC_*` vars are baked into `decolua/9router:latest` at image build time; runtime overrides only work via `BASE_URL`/`CLOUD_URL` on the server side (SAML/OIDC/settings).
- `CONTEXT7_API_KEY`, `FIRECRAWL_API_KEY`, `JIRA_*`, `METABASE_*`, `ROUTER9_GATEWAY_KEY` exist only in the origin pm2 env and are **not** read by the app — they are not migrated.

## 9router-api build / runtime quirks

The build and runtime hit several non-obvious issues that are handled inside `src/9router-api/Dockerfile` and the esbuild config. Do not "simplify" these without re-testing.

1. **`@/*` path aliases** — `server.ts` reaches into the `9router` source tree via TS aliases (`@/*` → `../9router/src/*`). The esbuild bundle keeps `9router` **external**, so at runtime Node loads raw 9router source containing `@/*` and extensionless imports Node cannot resolve natively.
2. **Node ESM loader** — the runner stage bakes an `alias-hook.mjs` registered via `--import` that rewrites `@/*` → `/app/9router/src/*`, `open-sse/*` → `/app/9router/open-sse/*`, `9router/*` → `/app/9router/*`, and resolves extensionless/`index.js` specifiers. Required for the external-9router strategy to run.
3. **Do NOT inline all of 9router** — removing `--external:9router` bundles `undici` and crashes at runtime with `Dynamic require of "node:assert" is not supported`.
4. **`require('./package.json')`** — the app reads its own version at runtime relative to `dist/`; the Dockerfile copies `package.json` into `dist/` so it resolves.
5. **Healthchecks bind IPv4 only** — use `127.0.0.1`, not `localhost` (busybox resolves `localhost` → `::1` and fails). `9router-api` health returns `{"status":"ok","mode":"api-only","version":"..."}`.

## Bring up the stack

```bash
ssh tencent-cloud
cd /opt/9router

# Both app services are built from the mirrored sources
# (src/9router and src/9router-api) that CI syncs to this host.
docker compose build 9router 9router-api
docker compose up -d
docker compose ps
```

Expected health: `9router`, `9router-api`, `caddy` all `(healthy)`. `cloudflared` is configured via `TUNNEL_TOKEN` from `env/cloudflared.env` (remotely-managed, automatically functioning).

## Cloudflare Tunnel

`cloudflared` runs in **remotely-managed** mode via `TUNNEL_TOKEN` (see `env/cloudflared.env`). Public hostnames are configured in the Cloudflare Zero Trust dashboard under tunnel `9router-tencent`, each mapping to `HTTP → caddy:80`:
- `9router-dashboard.vianhanif.link`
- `9routerapi.vianhanif.link`
- `9router.vianhanif.link`

Add/remove public hostnames there (dashboard-only; there is no API token for this repo). DNS CNAMEs for these hosts point at the tunnel and are proxied by Cloudflare.

## CI/CD (GitHub Actions)

`.github/workflows/deploy.yml` runs on push to `master` (and `repository_dispatch`). It scp's `docker-compose.yml`, `proxy/Caddyfile`, and `src/9router/` to Tencent, then runs `docker compose build 9router 9router-api` and brings the stack up.

Required repository secrets (in `vianhanif/9router-deploy` → Settings → Secrets → Actions):
- TENCENT_HOST=<your-server>
- TENCENT_USER=<deploy-user>
- `TENCENT_SSH_KEY` = private SSH key for the Tencent instance

> The env files and data are **not** deployed by CI (recursive gitignore). On a fresh host they must be provisioned once manually (see `env/` and `data/` above).
>
> Because `src/9router/` ships real upstream OAuth client credentials (already public), this repo should stay **private** — CI/deploy secrets also make it a poor candidate for public exposure.

## Cutover

`9router.vianhanif.link` and the dashboard/API hosts already point at the Tencent tunnel and serve through Caddy (verified live). The optional local pm2 stack (origin Mac) can remain stopped as standby; the API is now fully served from Tencent, independent of the laptop.
