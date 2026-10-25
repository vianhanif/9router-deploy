# 20261026-ROUTER-007-prod-parity

**Scope:** 9router-deploy parity with v0.5.109.

## Changes
- `src/9router/Dockerfile`: Pin node to `22.23.4-alpine`, add `ffmpeg`.
- `src/9router-api/Dockerfile`: Pin node to `22.23.4-alpine`.
- `docker-compose.yml`: Re-add `headroom` service (profiles: dashboard, networks: 9router-net).
- `docker-compose.yml`: Configure `9router` with `HEADROOM_URL` + `depends_on: headroom`.
- `.github/workflows/deploy.yml`: Add `headroom` to pull list.

## Validation
- `docker compose config` validates.
- `headroom` image pull requires auth (ensure `GHCR_PULL_TOKEN` is active).

## Risks
- Re-adding `headroom` (reversal of 2026-10-05 decision).
- Merge to `master` triggers deploy.yml (rebuilding images and restarting prod stack).
