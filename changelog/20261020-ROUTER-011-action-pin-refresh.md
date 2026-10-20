# 20261020-ROUTER-011 — Refresh all GitHub Action pins to current majors

**Status:** complete
**Scope:** `.github/workflows/*.yml` only.

## Why

None of the `uses:` pins had ever been updated, and dependabot was not watching
the `github-actions` ecosystem until ROUTER-009. By then the gaps were:

| Action | Was | Now |
|---|---|---|
| `actions/checkout` | v4 | v7 |
| `docker/build-push-action` | v6 | v7 |
| `docker/login-action` | v3 | v4 |
| `docker/setup-buildx-action` | v3 | v4 |
| `appleboy/ssh-action` | v1.0.3 | **v1.2.5** (already merged, #25) |
| `appleboy/scp-action` | v0.1.7 | **v1.0.0** (already merged, #25) |

## Compatibility checked before bumping

- **`docker/*` v4 and `build-push` v7 are Node 24 runtime bumps** that require
  Actions Runner v2.327.1+. GitHub-hosted `ubuntu-latest` satisfies this.
- **`setup-buildx` v4 removed deprecated inputs.** This repo uses that action
  with *no* `with:` block at all, so the removal cannot bite.
- **`build-push` v7 removed `DOCKER_BUILD_NO_SUMMARY` /
  `DOCKER_BUILD_EXPORT_RETENTION_DAYS`.** Neither is set anywhere in the repo.
- **`build-push` inputs in use** are `context`, `push`, `tags`, `build-args`,
  `cache-from` — all still supported; no removed input is referenced.
- **`appleboy/scp-action` v1.0.0** keeps `host`, `username`, `key`, `source`,
  `target`, which is the full set used here. (Verified against the v1.0.0
  README, since crossing pre-1.0 to 1.0 commonly renames inputs.)
- **`appleboy/ssh-action` v1.2.5** keeps `host`, `username`, `key`, `envs`,
  `script`, `command_timeout`, which is the full set used here.

## Why one PR instead of four

dependabot opened a PR per bump. Each touches `deploy.yml`, which is in its own
push-path filter, so merging them one at a time would have run **four sequential
production deploys** to change version strings. Consolidated into a single
change → one deploy, and one failure to attribute if something breaks.

It also unblocked dependabot: its `open-pull-requests-limit: 5` was saturated,
which is why no PR had been opened for `docker/build-push-action 6 → 7` even
though it was outdated. That bump is included here.

## Validated

The merge deploys the pinned release again (`v0.5.104` → `3c4ef419` / `1eaf229`),
exercising every bumped action in the real pipeline: `checkout` → `setup-buildx`
→ `login-action` → `build-push` (×2, via `build-test`) → `scp-action` → `ssh-action`.

A failure in any of them aborts before the prod switch, so the blast radius of a
bad bump is a red run rather than a broken deploy.
