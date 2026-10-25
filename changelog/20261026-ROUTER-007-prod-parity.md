# 20261026-ROUTER-007 — v0.5.109 prod parity (node pin, ffmpeg; headroom NOT re-added)

**Status:** prepared; **not merged** — merge triggers `deploy.yml` (prod deploy)
**Scope:** `9router-deploy` only. Upstream `9router` / `9router-api` repos untouched.

## Changes

- `src/9router/Dockerfile` — `ARG NODE_IMAGE=node:22-alpine` → `node:22.23.4-alpine` (all 3 stages); runner `apk add --no-cache su-exec` → `su-exec ffmpeg` (Codex STT needs ffmpeg). Multi-stage git-fetch design unchanged.
- `src/9router-api/Dockerfile` — `FROM node:22-alpine` → `node:22.23.4-alpine` in `ninesrc` + `apibuild`. Runner stays `node:22-bookworm-slim` (untouched).
- `.github/workflows/deploy.yml` — advance release pins: `PIN_9ROUTER: v0.5.104` → `v0.5.109`, `PIN_9ROUTER_API: v0.5.104` → `v0.5.109`.
- `docker-compose.yml` — **NO CHANGE**. Headroom service intentionally NOT re-added; measurement (20261005-REMED-remove-headroom.md) showed ~1.5% effective token reduction at 336 MiB RAM cost + 11% CPU — negligible vs. RTK core compression (22–68%). Costs exceed benefit with `headroomEnabled=false` (default); reclaimed.

## Validation

| Check | Result |
|---|---|
| `docker compose config` | Pass (parse + service graph) |
| `grep -c headroom docker-compose.yml` | 0 |
| `PIN_9ROUTER` + `PIN_9ROUTER_API` | both = v0.5.109 |
| Parity (node 22.23.4, ffmpeg) | Applied to both Dockerfiles; caddy / cloudflared / 9router-api untouched |

## Risks

1. **Merging this PR triggers `deploy.yml`.** Paths filter matches `.github/workflows/deploy.yml`, `src/**`. Live prod deploy: runner image rebuild from `PIN_9ROUTER=v0.5.109`, scp of compose + Dockerfiles to `/opt/9router`, VPS smoke test, `up -d --force-recreate`, rollback on failure.
2. **Tags `v0.5.109` must exist in both source repos before merge.** Deploy resolves `PIN_9ROUTER` and `PIN_9ROUTER_API` via git ls-remote; missing tags fail the `Resolve source SHAs` step.

## Follow-ups (not in this PR)

- Confirm both tags `v0.5.109` created on `9router` and `9router-api` repos (required before merge).
