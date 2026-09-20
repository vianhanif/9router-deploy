# Task: Offline image builds via GitHub Actions + GHCR

- **Date:** 2026-09-20
- **Ticket:** OFFLINE
- **Status:** Planned
- **Base commit:** `41f3db5` (`master`)
- **Branch suggestion:** `feature/offline-ghcr-build`

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
2. **VPS CPU architecture is `linux/amd64`.** No arch evidence exists anywhere in the repo (searched all compose/workflow/README/Dockerfile/cloudflared files for `uname`, `aarch64`, `arm64`, `amd64`, `x86_64`, `--platform`, `TARGETARCH` — zero hits; `git log` arch grep: only unrelated README commit). `ubuntu-latest` runners are amd64. **If the VPS is arm64 this plan is wrong — see Open Question 1 (blocking).**
3. **GHCR images live in this repo's own namespace** `ghcr.io/vianhanif/9router-deploy/*`. The deploy repo's `GITHUB_TOKEN` can push there (with `packages: write`); a token from this repo cannot push to another repo's namespace.
4. **Source repos `vianhanif/9router` and `vianhanif/9router-api` are reachable unauthenticated.** `deploy.yml:72/81` run `git ls-remote https://github.com/vianhanif/9router.git` with no token and CI works; the Dockerfiles fetch the same way (`src/9router/Dockerfile:12-16`, `src/9router-api/Dockerfile:10-14,26-30`) without auth. If those repos are private, today's CI would already fail — strong evidence they are public. (Unverified whether the VPS has a credential helper; the runner will hit GitHub fresh.)
5. **No build-time secrets exist.** The only build args are resolved public SHAs (`deploy.yml:85-88`); Dockerfile `ARG`s are repo URLs + version refs (`src/9router/Dockerfile:6-7`, `src/9router-api/Dockerfile:5-6,21-22`), no `--mount=type=secret`, no build-time login. Nothing needs to move to GitHub Secrets for the build itself.
6. VPS runs Docker Compose v2 (all workflows use `docker compose`).
7. Prod image set maps 1:1 to GHCR refs: `9router`, `9router-api`, `headroom`.
8. GHCR packages will be **public** unless the user decides otherwise (see Open Question 2 — default private breaks anonymous VPS pull).

## Impact Scope

| # | Scope | Repository | Complexity |
|---|-------|------------|------------|
| 1 | `.github/workflows/deploy.yml` — build moves to runner, VPS steps become pull/switch | 9router-deploy | High |
| 2 | `docker-compose.yml` — `image:` refs replace `build:` | 9router-deploy | Med |
| 3 | `docker-compose.local.yml` — add `build:` back for local dev | 9router-deploy | Low |
| 4 | `README.md` — CI/CD + architecture doc refresh | 9router-deploy | Low |
| 5 | Manual ops (no repo): VPS `uname -m` check, GHCR visibility decision, optional VPS `docker login`, first rollout | VPS | Med |
| 6 | `preview.yml` + `docker-compose.preview.yml` | 9router-deploy | **Untouched — out of scope** |

## Change Approach

### Step 0 — Blocking preflight (user/ops, before coding)

1. **Confirm VPS arch** (`uname -m`). amd64 → proceed. aarch64 → STOP and re-plan (Open Question 1).
2. **Choose GHCR package visibility.** Public → anonymous `docker compose pull` on VPS, zero credentials to manage. Private → VPS needs `docker login ghcr.io` with a `read:packages` PAT; that credential lives in the VPS user's docker config (written by `docker login`), not in this repo (env files are gitignored, `env/*.env`; the deploy workflow only syncs compose/Caddyfile/Dockerfiles, `deploy.yml:42`). Do not invent secret values; this is a user decision.

### Step 1 — `deploy.yml` restructure (High)

Single job retained; steps re-ordered around a runner-side build stage, then SSH pull/switch:

**Removed (moved off VPS):**
- "Build new images" step (`deploy.yml:45-88`) — the SSH script body: `docker tag ...:prev` capture (`62-64`), `resolve_sha` (`66-78`), `docker compose build --build-arg` (`85-88`).

**Kept, relocated onto the runner (new steps):**
- SHA resolution as a plain `run:` step with `GITHUB_OUTPUT` outputs, preserving dispatch-default behavior: `INPUT_NINEROUTER/INPUT_API` fall back to `master` when `github.event.inputs` is empty (`56-60` semantics — `repository_dispatch` delivers no `inputs`, so the fallback matters).
- `permissions: contents: read, packages: write` on the job (GHCR push). GHA cache save may additionally require `actions: write` — Open Question 4.
- `docker/setup-buildx-action@v3`, then `docker/build-push-action@v6` with `username: ${{ github.actor }}`, `password: ${{ secrets.GITHUB_TOKEN }}`:
  - **9router:** context `src/9router`, build-arg `NINEROUTER_VERSION=$NINEROUTER_SHA`, tags `ghcr.io/vianhanif/9router-deploy/9router:sha-${NINEROUTER_SHA}` **and** `:latest`, `push: true`, `cache-from: type=gha`, `cache-to: type=gha,mode=max`.
  - **9router-api + headroom:** **build once, tag 4 refs** — `docker-compose.yml:41-44` and `62-65` build the *same* `./src/9router-api` Dockerfile; `headroom` differs only by compose-level entrypoint/command (`46-47` vs `131-132`). buildx multi-tag: `.../9router-api:sha`, `.../9router-api:latest`, `.../headroom:sha`, `.../headroom:latest`. Halves build time vs today's two identical VPS builds. (Revisit if the Dockerfile splits — Open Question 6.)

**Changed (SSH steps):**
- "Sync deploy config" (`36-43`): **keep scp as-is** — the two Dockerfiles are unused by prod now but zero-cost and serve the manual-rebuild fallback; preview re-syncs its own copy (`preview.yml:136`). Minimal diff.
- "Switch to new images & verify" (`176-191`): prepend before `up -d`:
  - `docker tag ghcr.io/vianhanif/9router-deploy/9router:latest .../9router:prev` (+ api, headroom) — **before** pull, preserving today's local-retag rollback.
  - `docker compose pull` — for the profile-gated dashboard service (`docker-compose.yml:18`), use `docker compose --profile dashboard pull 9router 9router-api headroom` (Risk 7).
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

1. `uname -m` on VPS — gate for the whole plan.
2. Visibility decision; if private: create PAT `read:packages`, `docker login ghcr.io` as the user appleboy logs in as (`TENCENT_USER`, `deploy.yml:40`).
3. First deploy post-merge: verify `docker compose pull` fetches 3 images, `up -d --force-recreate` succeeds, smoke hits GHCR images, `docker image ls` shows ghcr refs, `/opt/9router` builder cache untouched, preview still builds on VPS.
4. Ops routine: GHCR sha-tag retention (Risk 4).

### Step 6 — Fallback decision: delete the on-VPS prod build path (YAGNI justified)

Do NOT keep a dual build path. Reasons:
- Rollback is already registry-free once images are local (`:prev` tag dance); today's `:prev`-before-build capture (`62-64`) becomes `:prev`-before-pull — same shape, no new machinery.
- An emergency manual rebuild is a 2-line `docker build` with identical build-args; the old script survives in git (`deploy.yml@41f3db5:45-88`).
- `preview.yml:203` keeps the VPS build machinery exercised, so it cannot bitrot.
- Keeping both paths forces every deploy to answer "which path built this image?" — an untestable branch that only fires during a failure.

## Risks & Side Effects

1. **Architecture mismatch (blocking).** amd64 images on an arm64 VPS = `exec format error` crash loops on every service. Verify `uname -m` before merge. If arm64: `--platform linux/arm64` via QEMU on amd64 runners (much slower; the manylinux `headroom-ai==0.37.0` wheel at `src/9router-api/Dockerfile:49` may lack aarch64 builds — unverified) or an arm64 self-hosted runner — material cost/benefit change.
2. **GHCR auth drift on VPS (private packages only).** PAT expiry → pull 401 → deploy gate aborts (prod untouched, rollback no-op). Mitigation: fail-fast + documented refresh. Public packages delete this risk class.
3. **Visibility misconfig.** GHCR defaults packages to private; a "public" plan with default settings 401s at VPS pull *after* a successful runner push. The failing step must log pull/auth vs image-content distinctly.
4. **GHCR storage/quota.** Every deploy adds sha tags (3 refs, incl. duplicate-content headroom). Private packages count against GHCR free storage (exact current limits unverified); public packages don't. `type=gha` cache keeps cache layers out of the registry (avoids `type=registry` doubling storage). Mitigation: ops routine deleting old sha tags, keeping `latest` + `prev`.
5. **Cache effectiveness.** `NINEROUTER_VERSION` busts the git-clone layer per deploy (by design, `src/9router/Dockerfile:10-12`); `npm install` layer hits while `package.json` is unchanged; the Next/esbuild build layer always re-runs. `type=gha` stores cache in the Actions cache service (per-repo 10GB free tier) — no registry bloat; eviction to verify (Open Questions 4-5).
6. **Smoke-ref drift.** Hardcoded local image refs across smoke (`119,129,148`), prune (`253`) and rollback (`283,289`) must all move to GHCR names in one change; any miss = smoke testing the stale image or pruning nothing.
7. **`compose pull` profile trap.** `9router` is profile-gated (`docker-compose.yml:18`); a bare `docker compose pull` may skip it and let `up` implicitly pull (or fail). Use `--profile dashboard` explicitly.
8. **Preview constraints only partly lifted.** Runner-side prod builds remove the concurrent-build OOM half of `791769f`'s rationale; the cloudflared-recreate-vs-edge-verify race survives (assessment below).

## Preview push-trigger assessment (commit `791769f`)

`791769f:3-6` cites two reasons preview stays dispatch-only: (a) concurrent prod+preview builds on the 1.7GB VPS (OOM), (b) a race between a prod deploy recreating cloudflared and the preview edge verify that rides prod cloudflared. Runner-side prod builds **remove (a)** — prod never builds on the VPS; preview's own build (`preview.yml:203`) then runs alone. **But (b) remains**: the concurrency groups are disjoint (`preview-9router-deploy`, `preview.yml:35` vs `deploy-9router`, `deploy.yml:26`), so a push trigger would still run preview + prod switches concurrently — both force-recreate members of the shared network (`docker-compose.preview.yml:24-26`) and both verify through the same prod cloudflared. Offloading builds is **necessary but not sufficient** for a push trigger; that change would additionally need a cross-workflow lock, a verify path independent of prod cloudflared, or a preview build memory cap. **Not changed in this PR** (scope).

## Unverified / Open Questions (blocking — need user/VPS input)

1. **VPS CPU architecture.** Zero repo evidence; plan assumes amd64. Run `uname -m` on the VPS before implementation. If aarch64: replan around QEMU cross-build or an arm64 runner.
2. **GHCR package visibility (public vs private) — user decision.** If private: where the VPS credential lives and which OS user's docker config receives `docker login` (appleboy SSH runs as `TENCENT_USER`, not necessarily root).
3. **Are `vianhanif/9router` / `vianhanif/9router-api` private?** Runner-side builds hit github.com from a fresh VM; if private, the Dockerfile `git fetch` needs a token the runner does not currently have (same-org `GITHUB_TOKEN` or PAT). Evidence says public; confirm.
4. **GH Actions cache write permission.** Whether `actions: write` is required for `cache-to: type=gha` on this repo (known 403 gotcha) — verify at implementation.
5. **GHCR storage limits + sha-tag retention policy.** Decide cleanup cadence; confirm whether public packages are exempt from the storage quota.
6. **Single image for headroom + 9router-api** (same Dockerfile/context, `docker-compose.yml:41-44,62-65`) — confirm acceptable long-term (entrypoint/command differ only in compose). If the Dockerfile ever splits, revisit.
7. **Exact `docker compose --profile dashboard pull <services>` behavior** and whether `up -d` re-pulls implicitly — verify at implementation (Risk 7).

## Out of Scope

- Any change to `preview.yml` or `docker-compose.preview.yml` (push trigger, preview pull-from-GHCR) — deliberately excluded from this PR.
- Container runtime/resource limits, TLS, ufw, hostname alignment, REMED leftovers.
- GHCR quota cleanup automation, JWT rotation, Kiro.