# 20261019-ROUTER-007 — GHCR offline build (production images built in Actions, VPS pulls)

**Status:** complete (branch `feature/ghcr-offline-build`, PR open); deploy activates on merge to `master`
**Scope:** 9router-deploy only. `9router` and `9router-api` source trees are untouched — their Dockerfiles stay in place and are still consumed by the runner builds (`context: src/...`) and by the VPS fallback path.

## Why

Production images were built **on the VPS** inside `deploy.yml`'s `docker build` step:

- **Resource exhaustion.** 1.7 GB RAM / 2 vCPU / ~30 GB disk box. Cold builds pushed it into swap and OOM-killed neighbouring containers on the shared daemon (the co-located `opencode-sandbox` stack).
- **Duplicated builds.** `deploy.yml` and `preview.yml` both rebuilt the same Dockerfile layers on the same weak box, and the VPS BuildKit cache had to be capped (`--max-used-space 3G`) just to stop runaway growth.
- **SSH timeouts.** The build ran inside `appleboy/ssh-action` with `command_timeout: 30m`. Cold multi-stage builds (npm + node-gyp + Next.js) routinely pushed past it, leaving half-built state on the box.

Moving the build to GitHub-hosted runners (2 vCPU / 7 GB, ephemeral disk, no neighbour containers) fixes all three at once, and `docker compose pull` on the VPS becomes a cheap layer download instead of a compile.

## What moved

| Before (master) | After (this branch) |
|---|---|
| `deploy.yml` builds `9router` + `9router-api` over SSH on the VPS | `deploy.yml` builds both images on the runner with `docker/build-push-action@v6` + `docker/setup-buildx-action@v3`, pushes `:latest` + `:sha-<sha>` to **private GHCR** (`ghcr.io/vianhanif/9router-deploy/{9router,9router-api}`) |
| `docker-compose.yml` uses `build:` for `9router` / `9router-api` | `image: ghcr.io/vianhanif/9router-deploy/{9router,9router-api}:latest` (pull-only) |
| VPS pull needs nothing (images built locally) | VPS logs in to GHCR with `GHCR_PULL_TOKEN` (classic PAT, `read:packages`), fails loudly if the secret is empty, logs out at the end of every run |
| `:prev` captured from freshly built `9router-9router:latest` | `:prev` captured **before** the pull, from the currently-running container's image (registry-independent rollback, even on first cutover) |
| smoke tests the just-built `:latest` | smoke tests the SHA-tagged refs (`:sha-<sha>`) so the implicit `docker run` pull does not overwrite local `:latest` before `:prev` is captured |
| — | new workflows: `build-test.yml` (runner build + boot-check, no VPS), `ghcr-pull-test.yml` (VPS pull + disk before/after), `cleanup-ghcr-tags.yml` (monthly retention sweep) |

`docker-compose.local.yml` keeps a `build:` override for both services so local development still builds from source instead of pulling prebuilt images.

## Private GHCR + `GHCR_PULL_TOKEN` decision

PR #8 planned **public** packages (anonymous pull, no credential on the box). This branch deliberately goes **private** instead:

- The images embed the deployment topology and pinned internal refs; keeping them private keeps that off the public registry.
- A pull-only classic PAT scopes the blast radius — the VPS can never push, and revoking the token kills VPS pull access without touching CI.
- Cost is nil either way (private storage/quota is not billed for Actions-built images in this usage pattern).

Consequence: **`GHCR_PULL_TOKEN` must exist before merge** — it is already present in the repo secrets (created for the Phase 1/2 validation runs).

## Rollback story

The full fail-safe chain works with **zero registry access**:

1. **Before** `docker compose pull`, the switch step captures the image the production container actually runs as `<ref>:prev`. On the first post-cutover deploy (no local ghcr `:latest` yet) it falls back to the pre-cutover compose-built name (`9router-9router:latest`) so rollback exists from day one.
2. If any step after the smoke test fails, the rollback step re-tags `:prev` → `:latest` and recreates the stack. It skips recreation entirely when `:latest` never diverged from `:prev` (build/pull failed before touching production), avoiding an ingress blip.
3. The failed pull's image body is removed by id (pulled GHCR images carry no `com.docker.compose.project` label, so the label-scoped dangling prune can never reclaim them), and `docker logout` clears the credential from the VPS at the end of every run.

`preview.yml` and `docker-compose.preview.yml` are unchanged — preview keeps its VPS-local build path.

## Preserved master behaviour (explicitly NOT regressed)

| Master-owned behaviour | Preserved because |
|---|---|
| headroom service removed (`20261005-REMED-remove-headroom.md`) | measured effective reduction was ~1.5% at 336 MB RAM cost — re-adding it would reclaim that regression. No `headroom` service, image, `HEADROOM_URL`, `depends_on`, or `smoke-headroom` anywhere on this branch |
| preview image purge (`9router-preview-*` rmi) in `preview.yml` | preview builds were a ~5 GB leak source; this branch does not touch `preview.yml` or `docker-compose.preview.yml` at all |
| BuildKit cache cap `--max-used-space 3G` + prune-on-failure (`if: !cancelled()`, after rollback) in `deploy.yml` | still needed for the VPS-local builds that `preview.yml` and manual rebuilds perform; the runner build uses `cache-to: type=gha,mode=max` instead |
| `cloudflared:2026.10.1` tag + `.github/dependabot.yml` | dependabot-managed; no reason to touch either |
| `deploy-9router` concurrency group, smoke → switch → verify order, project-scoped pruning (`label=com.docker.compose.project=9router`) | hard-won operational guarantees from prior incidents |

## Cutover plan (merge = deploy)

Merging this PR to `master` auto-triggers `deploy.yml` (push on `docker-compose.yml` matches its path filter). The first post-merge run is the cutover:

1. runner builds both images and pushes `:latest` to private GHCR (new package refs)
2. VPS logs in with `GHCR_PULL_TOKEN`
3. smoke test pulls the SHA-tagged refs (first GHCR pull on the box)
4. switch step captures `:prev` from the running pre-cutover images, pulls `:latest`, recreates the stack
5. verify + prune (legacy `9router-9router*:latest` compose-built refs are swept post-cutover)

Rollback for anything that fails after step 3 is automatic (local `:prev`). For a hard undo of the merge itself, revert the PR — the old compose `build:` path still works on the VPS via `docker-compose.local.yml`-style overrides.

## Testing

- `build-test.yml` (dispatch + push-triggered): builds both images on a runner, pushes `:test-ghcr-<sha>` / `:test-ghcr`, boots them on the runner, checks 9router `/api/health` == `{"ok":true}` and api `/api/health` contains `version`. Zero VPS contact.
- `ghcr-pull-test.yml` (dispatch): VPS pull of the `:test-ghcr` refs with disk before/after, assert-present, rmi cleanup.
- `deploy.yml` itself is only exercised by a real merge (no dry-run mode on the VPS path).

## Files touched

- `docker-compose.yml` — 9router / 9router-api `build:` → `image:` (GHCR refs)
- `docker-compose.local.yml` — `build:` overrides for local dev
- `.github/workflows/deploy.yml` — runner-side build & push; VPS becomes pull/switch; GHCR login/logout
- `.github/workflows/build-test.yml` — new, runner build + boot-check
- `.github/workflows/ghcr-pull-test.yml` — new, VPS pull test
- `.github/workflows/cleanup-ghcr-tags.yml` — new, monthly GHCR retention
- `README.md` — CI/CD docs refresh

## Corrections (2026-10-19, same day)

- **Packages are `public`, not private.** This document assumed private GHCR
  (500 MB free, metered). Checked after the cutover: `visibility=public` for both
  packages, and the repo itself is public. Public packages have free storage and
  free egress, and public repos get free Actions minutes — so none of the GHCR
  work above was a cost saving. `GHCR_PULL_TOKEN` is retained as a defensive
  fallback (anonymous pulls also work) rather than a requirement.
- **`cache-to: type=gha,mode=max` is gone.** It was removed before the cutover
  after `error writing layer blob: not_found` at 7.4 GB / 10 GB cache quota. The
  build reads cache only.
- **Retention is daily, not monthly**, with `RETENTION_DAYS=2` / `KEEP_NEWEST=2`
  and a `retention_days` dispatch override. Kept for tag-list hygiene; with the
  packages public it is not a billing control.
- **`resolve_sha` now accepts tags** (peeling annotated tags with `^{}`), so a
  deploy can pin `ninerouter_branch=<tag>` instead of tracking a branch tip.
