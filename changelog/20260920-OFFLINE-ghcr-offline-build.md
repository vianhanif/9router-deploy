# Task: Offline image builds via GitHub Actions + GHCR

- **Date:** 2026-09-20
- **Ticket:** OFFLINE
- **Status:** Planned — all 7 open questions resolved (see Resolutions). **Documentation only: no code, no workflow changes in this PR.**
- **Revision:** R2 (2026-09-20) — records resolutions of the 7 open questions (evidence-verified)
- **Base commit:** `41f3db5` (`master`)
- **Branch:** `feature/offline-ghcr-build` (draft PR #8)

## Task Overview

**What.** Move the production Docker image build off the VPS and onto GitHub-hosted runners: build → push to GHCR → VPS runs `docker compose pull` + `up -d`. Replaces the current on-VPS `docker compose build` in `.github/workflows/deploy.yml:85-88`.

**Why.** The VPS is ~1.7GB RAM and shared with production, the preview stack, and other projects (opencode-sandbox — see `changelog/20260915-REMED-9router-remediation.md:97-107`). On-VPS builds (a) consume RAM/CPU/disk on the production host, (b) run twice per branch tested (prod + preview), with preview explicitly kept dispatch-only to avoid concurrent-build OOM (`preview.yml:3-6`, commit `791769f`). GitHub-hosted runners are free-tier, amd64, dramatically larger than the VPS, and remove the 30m/60m SSH build timeouts (`deploy.yml:51`, `preview.yml:167`).

**Success criteria.**
1. A `master` push (or dispatch) builds all 3 prod images on `ubuntu-latest`, pushes them to GHCR, and the VPS pulls + starts them — `docker compose build` never executes on the VPS for the production path.
2. Existing orchestration preserved verbatim: `:prev` local-image rollback (`deploy.yml:264-305`), smoke test → switch → verify gate order, `deploy-9router` concurrency group (`deploy.yml:25-27`), project-scoped pruning.
3. Rollback requires no registry write access and no VPS build (works from already-pulled local images).
4. `preview.yml` and `docker-compose.preview.yml` untouched.

## Assumptions

1. **"Offline" = built OFF the VPS (out-of-band), NOT an air-gapped/isolated build.** No internet-disconnected environment is involved; builds happen on GitHub-hosted runners. The word "offline" only contrasts with today's on-box build. Anyone reading "offline" as "no network" is misreading the ticket.
2. **VPS CPU architecture is `linux/amd64` — RESOLVED (match).** VPS `uname -m` = `x86_64`; docker server platform `linux/amd64`; `ubuntu-latest` runner is x86_64. Architectures match: **no QEMU, no `--platform` override, no arm64 runner.** (`headroom-ai==0.37.0` publishes both `..._manylinux_2_28_aarch64.whl` and `..._manylinux_2_28_x86_64.whl`, so the `--only-binary :all:` install at `src/9router-api/Dockerfile:49` is not an arch blocker either way — detail in Resolutions Q1.)
3. **GHCR images live in this repo's own namespace** `ghcr.io/vianhanif/9router-deploy/*`. The deploy repo's `GITHUB_TOKEN` can push there (with `packages: write`); a token from this repo cannot push to another repo's namespace.
4. **Source repos `vianhanif/9router` and `vianhanif/9router-api` are public — RESOLVED (repo owner confirmed).** `deploy.yml:72/81` run `git ls-remote https://github.com/vianhanif/9router.git` unauthenticated today; the Dockerfiles fetch the same way (`src/9router/Dockerfile:12-16`, `src/9router-api/Dockerfile:10-14,26-30`) with no token. No token needed for the Dockerfiles' git fetch.
5. **No build-time secrets exist.** The only build args are resolved public SHAs (`deploy.yml:85-88`); Dockerfile `ARG`s are repo URLs + version refs (`src/9router/Dockerfile:6-7`, `src/9router-api/Dockerfile:5-6,21-22`), no `--mount=type=secret`, no build-time login. Nothing needs to move to GitHub Secrets for the build itself.
6. VPS runs Docker Compose v2 (all workflows use `docker compose`).
7. Prod image set maps 1:1 to GHCR refs: `9router`, `9router-api`, `headroom`.
8. **GHCR packages will be public — RESOLVED.** Public packages allow anonymous pull (VPS needs zero registry credentials) and are free (no quota). See Resolutions Q2.

## Impact Scope

| # | Scope | Repository | Complexity |
|---|-------|------------|------------|
| 1 | `.github/workflows/deploy.yml` — build moves to runner, VPS steps become pull/switch | 9router-deploy | High |
| 2 | `docker-compose.yml` — `image:` refs replace `build:` | 9router-deploy | Med |
| 3 | `docker-compose.local.yml` — add `build:` back for local dev | 9router-deploy | Low |
| 4 | `README.md` — CI/CD + architecture doc refresh | 9router-deploy | Low |
| 5 | Manual ops (no repo): first rollout (arch + visibility resolved — see Resolutions) | VPS | Low |
| 6 | `preview.yml` + `docker-compose.preview.yml` | 9router-deploy | **Untouched — out of scope** |

## Change Approach

### Step 0 — Preflight (RESOLVED — see Resolutions)

1. **VPS arch confirmed `x86_64`** (`uname -m`; docker server platform `linux/amd64`). Matches the `ubuntu-latest` x86_64 runner → no cross-build. (Q1)
2. **GHCR visibility decided: PUBLIC.** Anonymous `docker compose pull` on the VPS, zero credentials to manage. (Q2)
3. **Precondition before the FIRST push:** audit that no secret/token is baked into an image layer via build ARG or a copied env file — required because packages will be public. (Q2)

### Step 1 — `deploy.yml` restructure (High)

Single job retained; steps re-ordered around a runner-side build stage, then SSH pull/switch:

**Removed (moved off VPS):**
- "Build new images" step (`deploy.yml:45-88`) — the SSH script body: `docker tag ...:prev` capture (`62-64`), `resolve_sha` (`66-78`), `docker compose build --build-arg` (`85-88`).

**Kept, relocated onto the runner (new steps):**
- SHA resolution as a plain `run:` step with `GITHUB_OUTPUT` outputs, preserving dispatch-default behavior: `INPUT_NINEROUTER/INPUT_API` fall back to `master` when `github.event.inputs` is empty (`56-60` semantics — `repository_dispatch` delivers no `inputs`, so the fallback matters).
- **Add the job `permissions:` block (REQUIRED — verified gap, not a maybe).** Repo default workflow-token permission is `read` (`default_workflow_permissions: "read"`, verified via API); grep found no `permissions:` block in `deploy.yml`/`preview.yml`, so GHCR push and `cache-to: type=gha` would 403 without it (Q4):
  ```yaml
  permissions:
    contents: read
    packages: write      # GHCR push via GITHUB_TOKEN
    actions: write       # Actions cache API, only needed if cache-to: type=gha is used
  ```
- `docker/setup-buildx-action@v3`, then `docker/build-push-action@v6` with `username: ${{ github.actor }}`, `password: ${{ secrets.GITHUB_TOKEN }}`:
  - **9router:** context `src/9router`, build-arg `NINEROUTER_VERSION=$NINEROUTER_SHA`, tags `ghcr.io/vianhanif/9router-deploy/9router:sha-${NINEROUTER_SHA}` **and** `:latest`, `push: true`, `cache-from: type=gha`, `cache-to: type=gha,mode=max`.
  - **9router-api + headroom:** **build once, tag 4 refs** — `docker-compose.yml:41-44` and `62-65` build the *same* `./src/9router-api` Dockerfile; `headroom` differs only by compose-level entrypoint/command (`46-47` vs `131-132`). buildx multi-tag: `.../9router-api:sha`, `.../9router-api:latest`, `.../headroom:sha`, `.../headroom:latest`. Halves build time vs today's two identical VPS builds. (Revisit if the Dockerfile splits — Open Question 6.)

**Changed (SSH steps):**
- "Sync deploy config" (`36-43`): **keep scp as-is** — the two Dockerfiles are unused by prod now but zero-cost and serve the manual-rebuild fallback; preview re-syncs its own copy (`preview.yml:136`). Minimal diff.
- "Switch to new images & verify" (`176-191`): prepend before `up -d`:
  - `docker tag ghcr.io/vianhanif/9router-deploy/9router:latest .../9router:prev` (+ api, headroom) — **before** pull, preserving today's local-retag rollback.
  - `docker compose --profile dashboard pull 9router 9router-api headroom` — the resolved invocation form (Q7).
  - then unchanged `up -d --remove-orphans --force-recreate` (`189-191`).
- Smoke test (`90-174`): image refs → GHCR refs — `9router-headroom:latest` (`119`), `9router-9router:latest` (`129`), `9router-9router-api:latest` (`148`). Miss one and the smoke test silently tests the previous image.
- Prune (`245-262`):
  - `docker rmi ...:prev` refs (`253`) → GHCR refs.
  - **`docker image prune --filter label=com.docker.compose.project=9router` (`257-259`) stops matching** — compose applies that label to images it **builds**; pulled registry images carry no compose project label. Add `--filter "reference=ghcr.io/vianhanif/9router-deploy/*"` (keep `until=24h`).
  - **Keep `docker builder prune -f --filter until=72h` (`262`)** — still needed for preview builds (`preview.yml:203`) until preview migrates.
- Rollback (`264-305`): logic unchanged; swap the 3 svc refs in the `:prev` loop (`283`) + restore-tag (`289`). Registry-free: works from local images with GHCR down.
- Keep: concurrency group (`25-27`), smoke/verify gates, `if: failure()` rollback.

**Deploy gate behavior (fail-fast, keep old containers):**
- Build/push fails → job dies before any SSH step → zero VPS impact (strictly safer than today).
- Push OK, VPS pull fails (auth 401 / arch / network) → `set -e` aborts before `up`; old containers untouched; rollback finds `:latest`==`:prev` → no-op (`282-293` logic preserved).
- Partial pull (multi-service drop) → compose pull fails the step → no `up` → same no-op rollback.
- Pull+smoke OK, switch verify fails → unchanged today's path: rollback restores `:prev`.

### Step 2 — `docker-compose.yml` (Med)

- `9router` (`15-39`): `build: ./src/9router` (`16`) → `image: ghcr.io/vianhanif/9router-deploy/9router:latest`.
- `headroom` (`41-60`): `build:` block (`42-44`) → `image: ghcr.io/vianhanif/9router-deploy/headroom:latest`.
- `9router-api` (`62-84`): `build:` block (`63-65`) → `image: ghcr.io/vianhanif/9router-deploy/9router-api:latest`.
- `caddy` (`87`) / `cloudflared` (`99`): untouched — already `image:`-based.

Rejected alternative: keep `build:` *and* `image:` in base compose. It leaves a live on-VPS build path (a bare `up` without a preceding `pull` would rebuild on the VPS — regression trap) and blurs which path ran in every deploy. **Delete `build:`; do not keep both** (YAGNI, Step 6).

### Step 3 — `docker-compose.local.yml` (Low)

Add `build:` back for local dev only: `9router: build: ./src/9router`; headroom + 9router-api build blocks mirroring the removed base ones (context `./src/9router-api`, same Dockerfile). This file is the local-dev override (usage line `docker-compose.local.yml:6`), so the compose merge must yield build-instead-of-pull locally. Coder must verify merge behavior: `image:` (base) + `build:` (override) → `docker compose up` builds locally. Fallback if merge misbehaves: a `docker-compose.override.yml`; default to the existing local file (fewest files).

### Step 4 — `README.md` (Low)

Update architecture flow (`README.md:5-35`), infra layout (`43-54` — Dockerfiles still deployed, unused for prod build), preview/CI section (`~180-220`) and Security (`280-285`): builds run on GitHub runners, VPS pulls from GHCR; drop "builds on VPS" wording; document optional `docker login ghcr.io` when packages are private.

### Step 5 — Manual rollout (ops, after merge, NOT in this PR)

1. Audit that no secret/token is baked into an image layer — required before the first push because packages will be public (Q2 precondition).
2. First deploy post-merge: verify `docker compose pull` fetches 3 images, `up -d --force-recreate` succeeds, smoke hits GHCR images, `docker image ls` shows ghcr refs, `/opt/9router` builder cache untouched, preview still builds on VPS.
3. After first push: check package visibility in the GitHub Packages UI, flip to public before the first deploy (Risk 3).
4. Ops routine: GHCR sha-tag retention (the scheduled workflow from Q5).

### Step 6 — Fallback decision: delete the on-VPS prod build path (YAGNI justified)

Do NOT keep a dual build path. Reasons:
- Rollback is already registry-free once images are local (`:prev` tag dance); today's `:prev`-before-build capture (`62-64`) becomes `:prev`-before-pull — same shape, no new machinery.
- An emergency manual rebuild is a 2-line `docker build` with identical build-args; the old script survives in git (`deploy.yml@41f3db5:45-88`).
- `preview.yml:203` keeps the VPS build machinery exercised, so it cannot bitrot.
- Keeping both paths forces every deploy to answer "which path built this image?" — an untestable branch that only fires during a failure.

## Risks & Side Effects

1. **Architecture mismatch — RETIRED (Q1).** VPS is `x86_64`/`linux/amd64`, matching the x86_64 `ubuntu-latest` runner: no cross-build, no QEMU, no `--platform linux/arm64` override. This risk class no longer exists.
2. **GHCR auth drift on VPS — RETIRED (Q2).** Public packages allow anonymous pull; no PAT, no `docker login`, no credential lifecycle. This risk class no longer exists.
3. **Visibility misconfig.** GHCR defaults packages to private; public is set after the first push, so a "public" plan with default settings 401s at VPS pull *after* a successful runner push. Mitigation: check package visibility in the GitHub Packages UI after the first push, flip to public before the first deploy. Log pull/auth vs image-content errors distinctly.
4. **GHCR storage/quota.** Every deploy adds sha tags (3 refs, incl. duplicate-content headroom). PUBLIC packages are free (no quota) — the measured private-package math is exactly what forced the public decision (Q2). `type=gha` cache keeps cache layers out of the registry (avoids `type=registry` doubling storage). Mitigation: the sha-tag cleanup cadence in "GHCR sha-tag retention policy" (Q5).
5. **Cache effectiveness.** `NINEROUTER_VERSION` busts the git-clone layer per deploy (by design, `src/9router/Dockerfile:10-12`); `npm install` layer hits while `package.json` is unchanged; the Next/esbuild build layer always re-runs. `type=gha` stores cache in the Actions cache service (per-repo 10GB free tier) — no registry bloat; eviction still to verify (Q4 permissions resolved, Q5 retention cadence resolved).
6. **Smoke-ref drift.** Hardcoded local image refs across smoke (`119,129,148`), prune (`253`) and rollback (`283,289`) must all move to GHCR names in one change; any miss = smoke testing the stale image or pruning nothing.
7. **`compose pull` profile trap.** `9router` is profile-gated (`docker-compose.yml:18`); a bare `docker compose pull` may skip it. **Chosen form (Q7):** `docker compose --profile dashboard pull 9router 9router-api headroom` — implementation must verify this behaves as intended vs letting `up -d` re-pull.
8. **Preview constraints only partly lifted.** Runner-side prod builds remove the concurrent-build OOM half of `791769f`'s rationale; the cloudflared-recreate-vs-edge-verify race survives (assessment below).

## Preview push-trigger assessment (commit `791769f`)

`791769f:3-6` cites two reasons preview stays dispatch-only: (a) concurrent prod+preview builds on the 1.7GB VPS (OOM), (b) a race between a prod deploy recreating cloudflared and the preview edge verify that rides prod cloudflared. Runner-side prod builds **remove (a)** — prod never builds on the VPS; preview's own build (`preview.yml:203`) then runs alone. **But (b) remains**: the concurrency groups are disjoint (`preview-9router-deploy`, `preview.yml:35` vs `deploy-9router`, `deploy.yml:26`), so a push trigger would still run preview + prod switches concurrently — both force-recreate members of the shared network (`docker-compose.preview.yml:24-26`) and both verify through the same prod cloudflared. Offloading builds is **necessary but not sufficient** for a push trigger; that change would additionally need a cross-workflow lock, a verify path independent of prod cloudflared, or a preview build memory cap. **Not changed in this PR** (scope).

## GHCR sha-tag retention policy

**Policy (Q5 resolution):** retain `:latest` and `:prev`; sha/digest tags are audit-only (referenced in plan text/deploy logs, not retained long-term). Retention is a housekeeping/storage concern only — correctness never depends on sha tags existing, because rollback uses the local `:prev` tag and is registry-free.

**Mechanism:** a small scheduled workflow (`.github/workflows/cleanup-ghcr-tags.yml`, tentative name) that runs monthly (`schedule: cron`) + on-demand (`workflow_dispatch`). Deletes sha tags older than 7 days while explicitly preserving `:latest`, `:prev`, and any `pr-*` tags. Implementation detail: use the GitHub Packages API (`GET /orgs/{org}/packages/{package_type}/{package_name}/versions`, `DELETE /orgs/{org}/packages/{package_type}/{package_name}/versions/{version_id}`) with `GITHUB_TOKEN` (requires `packages: write`). Scope 3 packages: `9router`, `9router-api`, `headroom`.

## Resolutions (all 7 open questions RESOLVED, evidence-verified)

**Q1 — VPS CPU arch:** x86_64 (RESOLVED — match).
- VPS `uname -m` = `x86_64`; `docker info` platform = `linux/amd64`. GitHub Actions `ubuntu-latest` runner is x86_64. Architectures **match** → no QEMU, no `--platform` override, no arm64 runner. **Arch risk removed** from the plan — no longer a blocker or mitigation case.
- Corroborating detail: `headroom-ai==0.37.0` on PyPI publishes `headroom_ai-0.37.0-cp310-abi3-manylinux_2_28_aarch64.whl` AND `..._manylinux_2_28_x86_64.whl`. The `--only-binary :all:` install at `src/9router-api/Dockerfile:49` would have a wheel for both archs, so that constraint is not an arch blocker either way. Full validation of the on-VPS OOM premise: VPS RAM 1.7Gi total, 564Mi available; VPS disk `/dev/vda2` 40G, 23G used, 16G free (60%). Measured image sizes: `9router-9router:latest` 750MB; `9router-9router-api:latest` 2.2GB; `9router-headroom:latest` 2.2GB (same image as 9router-api). ~5.15GB per prod image set; `:prev` is a re-tag, not a copy, so no disk-use doubling.

**Q2 — GHCR visibility:** PUBLIC (RESOLVED).
- Rationale: (a) source repos `vianhanif/9router` and `vianhanif/9router-api` are public (Q3); (b) public packages allow anonymous pull → VPS needs zero registry credentials (no PAT, no `docker login`, no `~/.docker/config.json` write); (c) GitHub Packages usage is free for public packages; (d) **DECISIVE quantitative reason:** private packages bill against the plan's included quota (GitHub Free ~500MB storage / 1GB transfer per month; GitHub Team = 2GB storage per GitHub docs), while the measured images are 750MB + 2.2GB + 2.2GB. The 2.2GB image alone exceeds the Free storage allowance roughly 4×, so private is effectively blocked-or-billed from the first push.
- Consequences: VPS deploy step needs no auth; one fewer credential lifecycle to rotate; visibility remains reversible (package can be flipped public→private later).
- **Precondition (gate before the FIRST push):** audit that no secret/token is baked into an image layer via build ARG or a copied env file. Marginal disclosure is otherwise low because the images are built FROM already-public source.

**Q3 — Source repo visibility:** both public (RESOLVED, repo owner confirmed).
- `vianhanif/9router` and `vianhanif/9router-api` are public. Consistent with the fact that `git ls-remote https://github.com/vianhanif/9router.git` in `deploy.yml` works unauthenticated today. No token needed for the Dockerfiles' git fetch.

**Q4 — GitHub Actions permissions:** a required change (RESOLVED — verified gap, not a maybe).
- Verified via API: repo default workflow-token permission is `read` (`default_workflow_permissions: "read"`). Grep found NO `permissions:` block in `.github/workflows/deploy.yml` or `.github/workflows/preview.yml`. Therefore GHCR push would 403, and `cache-to: type=gha` would 403, unless `deploy.yml` gains an explicit block.
- **Required change — the single highest-confidence implementation gap found:**
  ```yaml
  permissions:
    contents: read
    packages: write      # GHCR push via GITHUB_TOKEN
    actions: write       # Actions cache API, only needed if cache-to: type=gha is used
  ```

**Q5 — GHCR sha-tag retention:** cleanup cadence (RESOLVED).
- Policy: retain `:latest` and `:prev`; sha/digest tags are AUDIT-ONLY (referenced in plan text/deploy logs, not retained long-term).
- Mechanism: a small scheduled workflow (`schedule:` monthly + `workflow_dispatch`) that deletes sha tags older than 7 days while explicitly preserving `:latest`, `:prev`, and any `pr-*` tags.
- Note: retention is a housekeeping/storage concern only; correctness never depends on sha tags existing, because rollback uses the local `:prev` tag and is registry-free.

**Q6 — Single image covering 9router-api + headroom:** proceed (RESOLVED).
- One image, multi-tagged, is accepted as the long-term shape. Revisit only if the Dockerfile is deliberately split later.
- Supporting detail: `docker-compose.yml:41-44` and `62-65` build the same Dockerfile/context and differ only in entrypoint/command.

**Q7 — Pull invocation:** explicit service list + profile (RESOLVED).
- Chosen form: `docker compose --profile dashboard pull 9router 9router-api headroom`.
- Record as an implementation-verification item (NOT a plan blocker): confirm this behaves as intended vs letting `up -d` re-pull; verify during implementation.

## Out of Scope

- Any change to `preview.yml` or `docker-compose.preview.yml` (push trigger, preview pull-from-GHCR) — deliberately excluded from this PR.
- Container runtime/resource limits, TLS, ufw, hostname alignment, REMED leftovers.
- GHCR quota cleanup automation, JWT rotation, Kiro.