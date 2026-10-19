# 20261019-ROUTER-008 — Deploy docs corrected, tag-based pinning

**Status:** complete
**Scope:** 9router-deploy only. No app code, no prod behaviour change.

## Why

Two things were actively wrong or missing:

1. **`AGENTS.md` described the retired architecture.** It claimed both apps are
   "built from source at deploy time" and told agents to smoke test "the
   `docker run` pattern in `deploy.yml`". After ROUTER-007 the runner builds and
   the VPS only pulls, so an agent following that file would reintroduce on-VPS
   builds. `README.md` likewise called GHCR "private" and claimed
   `cache-to: type=gha,mode=max`, which was removed.
2. **Deploys could only pin a branch or a raw SHA.** Merges auto-deploy the
   upstream master tips, so an unrelated merge ships whatever landed upstream
   since the last deploy. A human-readable pin (a tag) was not resolvable.

## What changed

- `AGENTS.md` — builds, validation and the "`src/` is not app source" trap stated
  correctly.
- `README.md` — public GHCR, no `cache-to`, tag input documented, retention
  cadence corrected.
- `deploy.yml` — `resolve_sha` resolves a tag as well as a branch, peeling
  annotated tags (`refs/tags/<t>^{}`) so the image build gets a commit, not a tag
  object. Branch names still win over identically-named tags, preserving prior
  behaviour. Input docs now say "branch, tag, or 40-char SHA".
- `cleanup-ghcr-tags.yml` — comment corrected: retention is hygiene, not billing,
  because the packages are public.

## Verified, not changed

The smoke test is already a hard gate. `deploy.yml` is a single job; the smoke
step (line ~161) runs `set -e` with no `continue-on-error`, so any failure aborts
the job before the switch step (~243). `rollback` carries `if: failure()` and
restores `:prev` → `:latest` from images already on the VPS. No change was
needed, so none was made.

## Tag pinning: state of the two source repos

| repo | local tags | on origin | HEAD | relation |
|---|---|---|---|---|
| `vianhanif/9router` | 91 | **0** | `3c4ef419` | 123 commits **past** `v0.5.104` (`87a74e1`) |
| `vianhanif/9router-api` | 0 | 0 | `1eaf229` | untagged, `package.json` 0.5.104 |

Consequences:

- Pinning to `v0.5.104` today would ship **older** 9router code than prod runs.
- Tags must be **pushed to origin** before `git ls-remote` can see them; the 91
  local 9router tags exist only in the local clone.
- `9router-api` has no tag history at all, so the first tag is a new decision.

## Test

`resolve_sha` exercised against the live remotes: `master` resolves
(`3c4ef419…`); `v0.5.104` and a nonsense ref fail loudly with
`ERROR: cannot resolve ref …` — correct, since no tags are pushed yet. Once a
tag is pushed the same call resolves without a code change.
