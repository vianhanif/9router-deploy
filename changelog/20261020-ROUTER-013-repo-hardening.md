# 20261020-ROUTER-013 — Close repo-level security and CI hardening gaps

**Status:** complete
**Scope:** GitHub repo settings (API) + `.github/workflows/*.yml`. No deploy-path
behaviour change.

An audit prompted by "all best practices completed?" found seven gaps. All closed.

## Repo settings (applied via API, no PR needed)

### `master` had no branch protection

`GET /branches/master/protection` returned **`404 Branch not protected`**. Every
policy in this repo — "never push directly to master", "feature branches + PRs
only" — was a convention with nothing enforcing it. A direct
`git push origin master` would have succeeded and shipped to production.

Now protected:

| Setting | Value | Why |
|---|---|---|
| Require PR before merging | yes | Enforces the feature-branch rule mechanically |
| Required approvals | **0** | Single maintainer — any higher count deadlocks every PR |
| Dismiss stale reviews | yes | Cheap correctness if a second reviewer appears |
| Allow force-push | **no** | Force-pushing `master` would orphan what a deploy shipped |
| Allow branch deletion | **no** | Deleting `master` on a public repo is unrecoverable |
| `enforce_admins` | false | Keeps an escape hatch for a genuine emergency |

**Deliberately not added: required status checks.** `deploy.yml` triggers on
`push` to `master`, so it only runs *after* a merge — it can never gate one.
Requiring it would permanently deadlock every PR. `build-test.yml` does run
pre-merge but is dispatch/branch-triggered, so it is not a reliable required
check either.

### Secret scanning, push protection, and Dependabot alerts were all `disabled`

This repo holds `TENCENT_SSH_KEY`, `GHCR_PULL_TOKEN`, and cloudflared
credentials. All three now enabled:

- **Push protection** — blocks a credential *before* it enters history. The only
  one of the three that prevents the incident rather than reporting it.
- **Secret scanning** — alerts on anything already present.
- **Dependabot security updates** — distinct from the version updates in
  `dependabot.yml`; this one reacts to advisories in the dependency graph.

All are free on public repos.

### `delete_branch_on_merge` was `false`

Merged PR branches accumulated unless deleted by hand. Now `true`.

## Workflow changes

### Over-granted `actions: write` removed

`deploy.yml` and `build-test.yml` both requested `actions: write`. Neither
writes Actions artifacts, caches, or state — it was carried over from when
`cache-to: type=gha` existed (removed in ROUTER-007). Reading a cache via
`cache-from` needs no scope at all.

Dropped from both. `deploy.yml` is the workflow holding production deploy
credentials, so this is the pair that matters most.

### No `timeout-minutes` anywhere

GitHub's default job timeout is **360 minutes**. A wedged `ssh-action` or a hung
`docker compose pull` on the VPS would have burned up to six hours of runner time
*while holding the `deploy-9router` concurrency lock* — so no subsequent deploy
could even queue behind it.

Added: **30 min** on `deploy` and `build-test` (runs are ~12 min), **30 min** on
`preview` (own VPS build), **15 min** on the two small jobs
(`cleanup-ghcr-tags`, `ghcr-pull-test`).

### `preview.yml` declared no `permissions:`

It inherited the repo default (`read`), which is correct, but every other
workflow declares explicitly. Now has an explicit `contents: read`, so it stops
silently depending on the repo default never changing.

### `cleanup-ghcr-tags.yml` had no concurrency group

A manual dispatch during the 03:00 cron run would double-purge the same package.
The API deletes are per-version, so two racing runs produce noisy 404s rather
than corruption — but serializing costs nothing. Group added,
`cancel-in-progress: false`.

## Checked and confirmed correct (not gaps)

- **No `npm` Dependabot ecosystem** — this repo has no `package.json`/lockfile.
  The app deps live in the source repos. Correctly absent.
- **`allowed_actions: all`, `sha_pinning_required: false`** — consistent with
  Dependabot managing the `uses:` pins. SHA-pinning every action would defeat the
  readable version comments the pins rely on.
- **Repo default workflow token permission is `read`** — already set correctly;
  this is why `deploy.yml`'s explicit `packages: write` is required at all.
- **`deploy.yml` / `build-test.yml` use `contents: read`, not `write`** — they
  never push to git.

## Trade-offs accepted

- Required-approvals count `0` means branch protection stops *accidental* direct
  pushes, not a determined maintainer. That is the intent.
- `enforce_admins: false` leaves an admin bypass. Intentional escape hatch; the
  cost is that the protection is not absolute.
- `preview.yml` still builds on the VPS and is untouched here.
