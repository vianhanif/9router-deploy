# 20261026-ROUTER-007 — v0.5.109 prod parity (node pin, ffmpeg, headroom)

**Status:** prepared; **not merged** — merge triggers `deploy.yml` (prod deploy)
**Scope:** `9router-deploy` only. Upstream `9router` / `9router-api` repos untouched.

## Changes

- `src/9router/Dockerfile` — `ARG NODE_IMAGE=node:22-alpine` → `node:22.23.4-alpine` (all 3 stages); runner `apk add --no-cache su-exec` → `su-exec ffmpeg` (Codex STT needs ffmpeg). Multi-stage git-fetch design unchanged.
- `src/9router-api/Dockerfile` — `FROM node:22-alpine` → `node:22.23.4-alpine` in `ninesrc` + `apibuild`. Runner stays `node:22-bookworm-slim` (untouched).
- `docker-compose.yml` — re-add `headroom` service (`ghcr.io/chopratejas/headroom:latest`, `profiles: ["dashboard"]`, `restart: unless-stopped`, `logging: *default-logging`, `networks: [9router-net]`, **no host port** — 9router reaches it by service name over `9router-net`); `9router` gets `HEADROOM_URL: http://headroom:8787` + `depends_on: headroom: {condition: service_started}`. caddy / cloudflared / 9router-api untouched.
- `.github/workflows/deploy.yml` — the VPS pull step enumerates services (`docker compose --profile dashboard pull 9router 9router-api`), which would skip `headroom`; added `headroom` to that list. No other workflow change.

## Validation

| Check | Result |
|---|---|
| `docker compose config` | Pass (parse + service graph; no docker CLI on the authoring host — validated the compose file directly) |
| `ghcr.io/chopratejas/headroom:latest` pullable | **public** — anonymous GHCR token + manifest = HTTP 200; index digest `sha256:50b85d8e320cfcdf1b38919bb7ae067b93ff7a8de0a93f05b2c1246370200d1c`; linux/amd64 present |
| deploy.yml path | `docker compose pull` (now incl. headroom) then `up -d --remove-orphans --force-recreate`; headroom is covered |

## Risks

1. **Reverses `20261005-REMED-remove-headroom.md`.** That removal was measurement-driven: ~1.5% mean outbound reduction (≤3.2%) at 336 MiB RAM (~19.7% of the VPS) + ~11% CPU, versus RTK in 9router core at 22–68%. Re-adding re-incurs that cost. `headroomEnabled` defaults `false` in 9router core, so the container is inert until enabled in the dashboard — but it runs and holds RAM.
2. **Unpinned third-party `latest`.** Mutable tag, no digest pin; silent upstream change on re-pull. Upstream v0.5.109 specifies the same tag.
3. **Merging this PR triggers `deploy.yml`.** Paths filter matches `docker-compose.yml`, `src/**`, `.github/workflows/deploy.yml`. That is a live prod deploy: runner image rebuild from `PIN_9ROUTER=v0.5.104`, scp of compose + Dockerfiles to `/opt/9router`, VPS smoke test, `up -d --force-recreate`, rollback on failure. This PR does **not** bump the pin — images stay v0.5.104 until the pin is advanced separately.

## Follow-ups (not in this PR)

- Advance `PIN_9ROUTER` / `PIN_9ROUTER_API` to `v0.5.109` (separate reviewed change).
- Consider pinning headroom by digest.
