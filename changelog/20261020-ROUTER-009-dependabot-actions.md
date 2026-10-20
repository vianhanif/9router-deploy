# 20261020-ROUTER-009 — Dependabot tracks GitHub Actions pins

**Status:** complete
**Scope:** `.github/dependabot.yml` only. No prod impact.

## Why

`.github/dependabot.yml` declared exactly one ecosystem, `docker-compose`
(which only watches the pinned cloudflared tag). Nothing watched the `uses:`
pins in `.github/workflows/`, so they rotted silently:

| Action | Pinned | Latest |
|---|---|---|
| `actions/checkout` | v4 | v7.0.1 |
| `docker/build-push-action` | v6 | v7.4.0 |
| `docker/login-action` | v3 | v4.6.0 |
| `docker/setup-buildx-action` | v3 | v4.4.1 |
| `appleboy/ssh-action` | v1.0.3 | v1.2.5 |
| `appleboy/scp-action` | v0.1.7 | v1.0.0 |

That failure mode is quiet until it is not: GitHub retires an action's Node
runtime and the workflow stops starting, which on this repo means deploys stop
starting.

## What changed

Added a `github-actions` ecosystem entry, weekly, PR-only, `limit: 5`.
Minor/patch bumps are grouped into one PR; majors arrive individually.

## What was considered and rejected

- **Hand-bumping the pins.** Rejected. The two appleboy actions carry the
  production path (VPS SSH + SCP), and `scp-action@0.1.7 → 1.0.0` crosses
  pre-1.0, where input names commonly change. Those need reading, not batching.
- **Grouping everything including majors.** Rejected; a 3-major
  `actions/checkout` jump should not ride in a batch with brand-new action
  majors touching the deploy path.
- **Deleting `build-test.yml` / `ghcr-pull-test.yml` as spent cutover
  scaffolding.** Rejected. Both are active, dispatch-only, and cost nothing;
  `build-test` is currently the only thing that builds a branch before merge.
- **Giving `build-test` its own buildx cache `scope=`.** Rejected after
  checking the Dockerfile: the app source is cloned *inside* the build
  (`git fetch --depth 1 origin $NINEROUTER_VERSION`), so the npm layer is
  already keyed per resolved commit and the two workflows do not collide.
  A separate scope would only have slowed builds.

## Known, not caused by this change

`build-test` failed once on the docs-only branch `fix/docs-and-tag-pinning`
(`39305998228`) with `Module not found: Can't resolve '@aws-sdk/credential-providers'`,
while `deploy` built the same commit `3c4ef419` successfully 11 minutes later.
The Dockerfile runs `npm install` rather than `npm ci`, so dependency
resolution is not lockfile-pinned and a transient registry/resolution flake is
the likely cause. Re-running the job is the workaround. Switching the
Dockerfile to `npm ci` would make it deterministic but is an app-repo change
and out of scope here.
