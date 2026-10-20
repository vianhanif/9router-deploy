# 20261020-ROUTER-013 — Close repo-level security and CI hardening gaps

**Status:** complete, after one self-inflicted outage (see "Correction" below)
**Scope:** GitHub repo settings (ruleset + security APIs) + `.github/workflows/*.yml`.

An audit prompted by "all best practices completed?" found seven gaps. Two of my
own audit claims were wrong and one fix broke three workflows; all are recorded
here rather than quietly corrected.

### Verified state (re-queried, not assumed)

| Item | Result |
|---|---|
| Secret scanning | `enabled` |
| Push protection | `enabled` |
| Dependabot security updates | `enabled` |
| `delete_branch_on_merge` | `true` |
| Ruleset `master` (23709088) | `active`, rules: `deletion`, `non_fast_forward`, **`pull_request`** |
| `actions: write` in any workflow | absent |
| Top-level `timeout-minutes` in any workflow | absent |
| `actionlint` on all five workflows | no errors |

## Repo settings (applied via API, no PR needed)

### `master` had no branch protection

`GET /branches/master/protection` returned **`404 Branch not protected`**. Every
policy in this repo — "never push directly to master", "feature branches + PRs
only" — was a convention with nothing enforcing it. A direct
`git push origin master` would have succeeded and shipped to production.

**This repo uses a ruleset, not classic branch protection.** Classic protection
returns 404 regardless; the API that governs `master` is:

```
GET /repos/vianhanif/9router-deploy/rulesets
→ id=23709088 name=master target=branch enforcement=active
  rules: deletion, non_fast_forward
  conditions: ref_name.include = ["~DEFAULT_BRANCH"]
  bypass_actors: []
```

So deletion and force-push on `master` were **already blocked** before this work
— a real safeguard that was there and that I initially reported as missing,
because I queried the wrong endpoint (the classic-protection API, whose 404 I
misread as "nothing is protected").

**What was actually missing is "require a pull request".** Added by editing
ruleset `23709088` to include the `pull_request` rule (a `PUT` to
`/rulesets/23709088`, not the classic protection endpoint). The ruleset now
carries `deletion`, `non_fast_forward`, and `pull_request`.

| Rule | Present before | Now |
|---|---|---|
| Block branch deletion | yes | yes |
| Block force-push (`non_fast_forward`) | yes | yes |
| Require a pull request | **no** | **yes** |
| Required approvals | n/a | **0** — single maintainer, any higher count deadlocks every PR |
| Bypass actors | none | none |

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

Added, all as **job-level** keys: **30 min** on `deploy`, `build-test`'s `build`
job and `preview`; **15 min** on `build-test`'s `boot-check` and
`cleanup-ghcr-tags`; **10 min** on `build-test`'s `resolve`.

#### Correction — the first attempt broke three workflows

The first version of this change put `timeout-minutes` at the **workflow top
level** in `deploy.yml`, `build-test.yml`, and `preview.yml`. It is not a
workflow-level key. GitHub rejects the entire file:

> `This run likely failed because of a workflow file issue.`

Run `39439088344` failed for exactly this reason. It was triggered by the branch
push (the `deploy.yml` push filter matches any branch), not by a master deploy —
but the same file would have been broken on master had it merged.

**Two process failures allowed this through, both worth fixing permanently:**

1. **"YAML validated" was not validation.** `ruby -ryaml` checks well-formedness,
   not schema. All five files parsed fine while three were invalid GitHub
   workflows. Syntax checks were reported as validity checks.
2. **A success signal was fabricated.** In the same audit, the branch-protection
   `PUT` returned an empty body and the script printed `" -> protected"` from its
   own `echo`, not from the API. Re-querying showed the protection had **never
   applied**. Assuming success from the absence of an error message.

**Fix:** `actionlint` is now the validator (`brew install actionlint` — was not
installed). It reports this precisely:

```
unexpected key "timeout-minutes" for "workflow" section.
expected one of "concurrency", "defaults", "env", "jobs", "name", "on",
"permissions", "run-name"
```

For any workflow change, `actionlint .github/workflows/*.yml` is the gate. It
also catches the `uses:`/input mistakes that a YAML parser never would.

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
