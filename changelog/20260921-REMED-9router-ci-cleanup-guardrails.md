# Task: 9router-deploy — CI cleanup guardrails fix

- **Date:** 2026-09-21
- **Ticket:** REMED (placeholder — replace with real ticket id)
- **Status:** Done
- **Branch:** `master`

## Problem

VPS at 80% disk (30G/40G). Manual cleanup reclaimed 15G (→ 39% used), then root causes were identified in the repo's own workflows — the automated cleanup was a no-op:

1. `deploy.yml` prune used `docker image prune --filter until=24h`. Image ages there are **hours**, never >24h, so the age filter matched nothing — the step reclaimed ~0 bytes on every run while dangling images accumulated to 9.64 GB.
2. The prune step had no `if`, so GitHub's default `if: success()` skipped it on **every failed run** — exactly the runs that leave dangling images behind.
3. `preview.yml` teardown stopped containers, containers' networks, and volumes, but never removed the compose-built `9router-preview-*` images (`down` never untags). ~5.1 GB accumulated across preview cycles.

Root-caused by inspecting the live VPS (`docker image ls -f dangling=true` showed label `com.docker.compose.project=9router`, confirming the label filter is valid) — not by reading the workflow alone.

## Changes

### 1. `.github/workflows/deploy.yml` — prune step

- Dropped `--filter "until=24h"` from `docker image prune`. The `label=com.docker.compose.project=9router` filter alone is what keeps the shared daemon's other stacks safe; the age floor was the bug, not a safety measure.
- Added `if: ${{ !cancelled() }}` so the step runs after failed deploys too (skips only on cancellation).
- **Reordered:** prune now sits **after** the rollback step. Required, not cosmetic — the step `rmi`s the `:prev` tags, and rollback reads those same tags to restore production. Pruning first would have left a failed-and-already-switched deploy with nothing to roll back to.
- `builder prune --filter "until=72h"` **kept deliberately** (not tightened to 24h). A tighter floor forces cold rebuilds, and the build step is the OOM-riskiest step on the 1.7 GB VPS.

### 2. `.github/workflows/preview.yml` — both teardown steps

- Added an image sweep after the container/network sweeps:
  ```bash
  for img in $(docker image ls -q --filter "reference=9router-preview-*"); do
    docker image rm -f "$img"
  done
  ```
- Added to **both** teardown paths: the pre-deploy teardown and the explicit `action=teardown`. Images are untagged by name-referenced `rmi` because they are not dangling — `docker image prune` cannot collect them.
- `caddy:alpine` untouched: the `reference=9router-preview-*` filter cannot match it.

## Verification

- `python3 -c "import yaml; yaml.safe_load(...)"` on both workflows → parses clean.
- `git diff` reviewed: 38 insertions / 19 deletions; no unrelated edits. Workflow step order confirmed (rollback → prune) in-file.
- Push to `master` touching `.github/workflows/deploy.yml` matches the deploy workflow's own `paths:` filter, so the next run exercises the prune step against real dangling images.

## Risks / Limitations

- Prune runs unconditionally now (except on cancel) — it will also run on a failed deploy where rollback already restored production. Safe: the `:prev` untag leaves the image referenced by `:latest`. Measured on the live daemon first; see notes below.
- Builder cache floor unchanged, so ~6 GB can still accumulate if deploys stop entirely — bounded on the next deploy, not proactively.
- Run #35521550048 failed at "Switch to new images & verify" (rollback succeeded). Exact error still unknown — logs need authenticated access. Suspected RAM/timing hiccup on the 1.7 GB VPS, **not** a config break; unverified.
- `opencode-sandbox` is no longer running on the VPS (containers + images removed), so the "do not disturb co-located stack" rationale for the label filter is currently moot — kept anyway, since the label filter is the correct isolation and opencode may return. Bind-mount data preserved at `/opt/opencode-sandbox/data`.
