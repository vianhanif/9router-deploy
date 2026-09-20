# Task: 9router-deploy Remediation

- **Date:** 2026-09-15
- **Ticket:** REMED (placeholder — replace with real ticket id)
- **Status:** Planned
- **Branch suggestion:** `remediation/20260915-hardening`

## Problem

Review of 9router-deploy vs live VPS found 6 remediation items:

1. Kiro AWS token-refresh noise (5-min scheduler, `invalid_grant` every cycle)
2. 10.4 GB reclaimable disk (2 dangling images + 5.97 GB build cache)
3. Unbounded docker logs (json-file, no rotation on all 5 services)
4. Floating `cloudflare/cloudflared:latest` tag (deployed 2026.8.3, latest 2026.9.1)
5. Unrotated `JWT_SECRET` shared between 9router + 9router-api env
6. Orphaned `JWT_SECRET` in 9router-api env (functionally unused)

## Scope

Approved scope (user-confirmed):

- `INITIAL_PASSWORD` — **SKIP** (already changed by user via 9router dashboard config)
- Kiro — env kill-switch only; code removal deferred
- JWT rotation — **documentation first**, then manual execution
- Prune — manual commands + deploy workflow guardrail
- Log rotation — compose `x-logging` anchor
- cloudflared — version tag pin

## Task Checklist

Workstream A — **Repo edits** (tracked, CI-synced; branch `remediation/20260915-hardening`):

- [ ] T1. Create `docs/jwt-rotation.md` — rotation runbook incl. load path (env → `data/9router/jwt-secret` fallback), HS256/24h single-secret, rotate via `openssl rand -hex 32` + `--force-recreate 9router`, verification + rollback, backup advice
- [ ] T2. Create `docs/remediation.md` — runbook: project-scoped prune commands, JWT rotation pointer, verification checklist (opencode-sandbox untouched), rollback
- [x] T3. `docker-compose.yml` — pin `cloudflare/cloudflared:2026.9.1` (avoid 2026.8.1)
- [x] T4. `docker-compose.yml` — add `x-logging` anchor (10m × 3, compress) + apply `logging: *default-logging` to all 5 services (9router, 9router-api, headroom, caddy, cloudflared)
- [x] T5. `.github/workflows/deploy.yml` — post-build prune guardrail (project-scoped image prune + `builder prune --filter until=72h`)

Workstream B — **VPS/manual** (gitignored env files + live server ops, NOT CI-synced):

- [ ] T6. `env/9router.env` (hub + VPS copy) — add `DISABLE_BACKGROUND_TOKEN_REFRESH=1` (Kiro scheduler kill-switch)
- [ ] T7. `env/9router-api.env` (hub + VPS copy) — optional same flag (API doesn't run scheduler; harmless)
- [ ] T8. Live prune on VPS — project-scoped only (`label=com.docker.compose.project=9router`); pre-check `docker system df` + `docker image ls -f dangling=true`; NEVER unfiltered `docker image/system prune -f`
- [ ] T9. JWT rotation manual execution — only after T1 doc reviewed/approved; backup `/opt/9router/data` first; expect one re-login wave
- [ ] T10. Post-deploy verification — no Kiro refresh errors, `/api/health` + `/v1/responses` OK, LogConfig max-size on all containers, cloudflared tag = 2026.9.1, disk ~27–30%, opencode-sandbox untouched

Deferred (separate tickets / explicitly skipped):

- [ ] D1. Remove `refreshKiroToken` AWS branch code in 9router repo (`KIRO_DISABLE_AWS_REFRESH` guard) — separate PR
- [ ] D2. Hostname alignment (3 different API hostnames)
- [ ] D3. ufw / ports
- [ ] D4. Orphaned `jira-mcp.env`
- [ ] D5. Resource limits
- [ ] D6. `INITIAL_PASSWORD` — **SKIPPED** (already changed by user via 9router dashboard config)

## Changes

### 1. Docs first — `docs/jwt-rotation.md`

Document before executing:

- How `JWT_SECRET` is loaded: env → persisted file fallback (`data/9router/jwt-secret`, dormant since env dominates)
- Single secret only (HS256, 24h expiry) — no dual-secret support in code
- Rotation Option 1 (recommended): `openssl rand -hex 32` → update `env/9router.env` → `docker compose --profile dashboard up -d --force-recreate 9router`
- Only 9router signs/verifies; api's `JWT_SECRET` functionally unused (docs-consistency only)
- Verification commands; rollback (restore old value + recreate)
- Expect one re-login wave; zero API/LLM downtime
- Backup `/opt/9router/data` before rotating

### 2. `docker-compose.yml`

- Pin `cloudflare/cloudflared:2026.9.1` (current latest stable; all dep CVEs fixed; avoid 2026.8.1)
- Add `x-logging` anchor:

```yaml
x-logging: &default-logging
  driver: json-file
  options:
    max-size: "10m"
    max-file: "3"
    compress: "true"
```

- Apply `logging: *default-logging` to all 5 services (9router, 9router-api, headroom, caddy, cloudflared)

### 3. `env/9router.env`

- Add `DISABLE_BACKGROUND_TOKEN_REFRESH=1` (Kiro scheduler kill-switch — exists in code, reversible)
- Optionally add same to `env/9router-api.env` for safety (API does not run the scheduler, but harmless)
- Note: env files are gitignored — this change is manual on VPS (and local copy), NOT synced by CI

### 4. `docs/remediation.md`

Runbook covering:

- Prune — **SCOPED to 9router project only** (VPS also runs opencode-sandbox + others):

```bash
# dangling images from ALL projects would be removed by unfiltered `docker image prune -f`
# — NEVER run unfiltered `docker image prune -f` or `docker system prune -f` on this VPS
# Remove only 9router-project dangling images (includes the 2 orphaned build intermediates):
docker image prune -f --filter "label=com.docker.compose.project=9router"
# Build cache is shared/not project-scoped; safe to clear but confirm no other stack is mid-build.
# Floor = 72h (decided 2026-09-20): deploys can be ~6 days apart, and CI retries land hours
# after a failed run — 24h would force a cold rebuild on every retry, while 72h still reclaims
# effectively all cache at steady state (REMED's 24h rec over-indexed on reclaim speed).
docker builder prune -f --filter until=72h
# Alternative pre-check: `docker system df` and `docker image ls -f dangling=true` before any prune
```

- JWT rotation execution steps (link to jwt-rotation.md)
- Verification checklist (confirm opencode-sandbox containers/images untouched after prune)
- Rollback procedures

### 5. `.github/workflows/deploy.yml`

- Add post-build prune guardrail:

```bash
docker image prune -f --filter "label=com.docker.compose.project=9router"
docker builder prune -f --filter until=72h
```

## Validation

- `docker logs 9router` — no new Kiro refresh errors after deploy
- `/api/health` + `/v1/responses` OK
- `docker inspect` — LogConfig max-size set on all containers
- Image tag = `cloudflared:2026.9.1`
- Disk after prune: ~27–30% used (from 54%)
- **opencode-sandbox (and other stacks) containers/images untouched** — `docker ps` for opencode stack unchanged
- JWT rotation manual step: login + dashboard loads after `--force-recreate`

## Out of scope / deferred

- Code removal of `refreshKiroToken` AWS branch in 9router repo (`KIRO_DISABLE_AWS_REFRESH` guard) — separate PR
- Hostname alignment (3 different API hostnames), ufw/ports, orphaned `jira-mcp.env`, resource limits — separate tickets