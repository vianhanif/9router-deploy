# 20261020-ROUTER-010 — Deploys default to a pinned release tag

**Status:** complete
**Scope:** `9router-deploy` (`deploy.yml`, `README.md`, `AGENTS.md`) plus one tag pushed to each source repo. No application code.

## Why

`deploy.yml` defaulted `ninerouter_branch` and `api_branch` to `master`, and the
`repository_dispatch` path fell back to `master` as well. The two source repos
are independent, so a merge in *either* one shipped that repo's new tip on the
next deploy — including a deploy triggered by an unrelated change in the other
repo. Merge and ship were the same action, with no review gate between them.

## What changed

`deploy.yml` now carries the pin as a single source of truth:

```yaml
env:
  PIN_9ROUTER: v0.5.104
  PIN_9ROUTER_API: v0.5.104
```

Resolution is `${INPUT_<X>:-$PIN_<X>}` — so a blank `workflow_dispatch` and
every `repository_dispatch` deploy the pinned release, while an explicit
`-f ninerouter_branch=<tag|branch|sha>` still overrides it.

Sources for those tags:

| repo | tag | peeled commit | was live? |
|---|---|---|---|
| `vianhanif/9router` | `v0.5.104` | `3c4ef419` | yes |
| `vianhanif/9router-api` | `v0.5.104` | `1eaf229` | yes |

## The tag-name decision

The fork's `v0.5.104` **moved forward** from upstream's commit `87a74e1b` to the
fork's own master `3c4ef419` — 123 commits, all fork work (upstream merges v0.5.99
and v0.5.104, the `/v1` Responses SSE fixes, opencode V2 config emission, tool
schema ranking, model discovery, P0 instrumentation).

That was deliberate. `package.json` at `3c4ef419` already declared `0.5.104`, so
the tag matches the version the code claims. The rejected alternative — pinning
to upstream's `v0.5.104` — would have deployed code *older* than production and
silently reverted all 123 commits. Upstream's own tag is untouched in upstream.

Consequence to remember: the fork's `v0.5.104` and upstream's `v0.5.104` are
different commits. Diffing the fork against upstream must use the upstream
remote, not the local tag.

## Validated

Deploy `39411289520` (the merge that introduced the pin) resolved

```
Deploying: 9router=v0.5.104 (3c4ef419630b2edb95e9d7368cd4ad8c824208ce), 9router-api=v0.5.104 (1eaf22957c33beb25f0da3eed12fa4f24dc58092)
```

with `docker compose build` absent from the log (VPS pull only), `DEPLOY OK`, and
prod on `0.5.104` with the dashboard healthy. Because both tags point at the
then-live commits, that first pinned deploy was a no-op — the safest available
end-to-end test of the pin path.

## Bumping the pin

1. Cut the release in the source repo and push the tag (`git tag -a`, then
   `git push origin <tag>`). Annotated tags are fine; `resolve_sha` peels them.
2. Update `PIN_9ROUTER` / `PIN_9ROUTER_API` in a reviewed PR here.
3. Merging that PR triggers the deploy.

Do **not** move an existing tag forward — that rewrites what previous deploys
shipped and destroys the tag's meaning.

## Trade-off accepted

The old failure mode was "unreviewed code ships on an unrelated merge". The new
one is "reviewed code sits unshipped if nobody advances the pin". The second is
preferable, but it is a new habit: after merging application fixes, the pin must
be advanced or production does not change.
