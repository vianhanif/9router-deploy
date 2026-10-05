# 20261005-REMED — Remove headroom token-compression proxy

**Status:** complete (repo changes); deploy pending push to `master`
**Scope:** 9router-deploy only. `9router` core and `9router-api` source are untouched — the headroom code paths there stay dormant (both are gated off by settings/env).

## Why

Measured from live logs (`9r-logs`, 2026-10-05, 10m36s / ~40 requests):

| Metric | Observed |
|---|---|
| headroom **effective** outbound reduction | **0.0–3.2%, mean ~1.5%** |
| headroom **reported** token delta | 0.0–6.5%, mean ~3.0% — **~2× inflated vs actual bytes** |
| calls that did nothing (`delta=0`, body byte-identical) | **~40%** |
| `tools` block | `293494B→293494B` on **every** call — never compressed |
| headroom's own log | `compression_ratio=1.0 files_total=0 hunks_total=0` (no-op) |
| headroom log content | ~397 of 400 lines = LiteLLM "Provider List" spam (no provider configured) |
| cost | **336.5 MiB RAM (19.7% of VPS) + ~11% of 2 vCPU** |

The useful compression is already done by **RTK in `9router` core** (`open-sse/rtk/`) at **22–68%** on `find`/`ls`/`grep`/`dedup-log` outputs, at zero extra RAM. With input cache reuse already at 70–99%, headroom's marginal saving was ~nil.

`headroomEnabled` already defaults to `false` (`9router` `settingsRepo.js:53`) and the user had disabled it in the dashboard — so compression requests had already stopped; the container was simply idle at 336 MB. This change reclaims that.

## Changes

- `docker-compose.yml` — removed the `headroom` service, its `HEADROOM_URL` env on `9router`, and the `depends_on: headroom` edge.
- `docker-compose.preview.yml` — removed `headroom-test` service, its `HEADROOM_URL`, and the `depends_on`.
- `.github/workflows/deploy.yml` — removed headroom from image tag/`build` targets, the `smoke-headroom` container, rollback `:prev` loop, and the prune `rmi`.
- `.github/workflows/preview.yml` — removed `headroom-test` from teardown, build, and start service lists.
- `src/9router-api/Dockerfile` — removed the `pyheadroom` stage (746 MB `headroom-ai` venv), the `HEADROOM_PYTHON` env, and the runner's `python3` apt install. **Shrinks the shared 9router-api image from 2.21 GB.**
- `env/9router-api.env.example` — `HEADROOM_COMPRESS_SHIM=off` (the shim defaults **on** when unset, so this must be explicit, not deleted).
- `README.md` — dropped headroom from the architecture diagram, services table, local-dev note, and prod/preview container lists.

## Deploy path

Push to `master` triggers `.github/workflows/deploy.yml`:
1. syncs `docker-compose.yml` + Dockerfiles to `/opt/9router`
2. rebuilds `9router` + `9router-api` (image shrinks ~746 MB)
3. smoke tests on scratch ports 20327/20328 — **now with no headroom dependency**
4. `docker compose --profile dashboard up -d --remove-orphans --force-recreate` → **reaps the `headroom` container as an orphan**
5. prunes; automatic rollback on failure

## Manual follow-up (not covered by the repo — env files are gitignored)

Set on the VPS in `/opt/9router/env/9router-api.env`:
```
HEADROOM_COMPRESS_SHIM=off
```
Without this, 9router-api defaults the shim **on** and would attempt to spawn the Python worker (which no longer ships in the image → fail-open + warning log).

## Expected result

- `headroom` container gone; `9router-api` ~336 MiB lighter; `MemAvailable` roughly triples (557 MB → ~890 MB).
- ~3 GB disk reclaimed (736 MB venv + 2.21 GB headroom image + build cache).
- Token counts rise ~1–3% (the RTK savings are unaffected; RTK is in 9router core).

## Verification

```bash
ssh tencent-cloud
docker compose -C /opt/9router ps         # expect 4 services, no headroom
free -h                                   # MemAvailable up ~336MB
docker images | grep headroom             # expect no 9router-headroom image
docker logs 9router-api --tail 200 | grep -i headroom   # expect shim-disabled lines only
```

Re-run the P0 baseline capture and compare `effective=` totals; if token counts rise materially more than ~3%, re-open this decision.

## Risks / follow-ups

- `9router` core and `9router-api` still contain headroom code (`open-sse/rtk/headroom.js`, `src/headroomCompressShim.ts`, `src/lib/headroom/*`). Left in place deliberately — dormant and gated. A future cleanup PR could remove them, but that touches two other repos and is out of scope here.
- If headroom is ever wanted again, it now requires reverting this commit (Dockerfile venv + service) — it is no longer merely "enable in dashboard".
- **Do not test this in preview**: `docker-compose.preview.yml` joins the prod network and has no rollback.
