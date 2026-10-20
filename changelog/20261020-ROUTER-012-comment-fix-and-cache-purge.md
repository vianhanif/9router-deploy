# 20261020-ROUTER-012 — Correct the "private GHCR" comment; purge the Actions cache

**Status:** complete
**Scope:** one comment in `deploy.yml`; one destructive-but-safe cache purge. No
workflow behaviour change.

## 1. Stale "packages are private" comment

`deploy.yml` claimed the GHCR packages are private and that the VPS therefore
*must* authenticate before any pull. Both halves are wrong: the packages are
public (verified `visibility=public`, anonymous pulls work), and the step is a
fallback rather than a requirement.

Rewrote it to say the packages are public, that the login is kept so the VPS
still deploys if visibility is ever flipped, and that it fails loudly when
`GHCR_PULL_TOKEN` is empty instead of silently depending on anonymity. The
ordering rationale (authenticate before any pull, credential persists in
`/root/.docker/config.json` for later `ssh-action` steps) was kept — that part
was correct. The `uses:` pin and step definition are untouched.

## 2. Actions cache purge — 8.3 GiB across 168 entries

**What it was:** every entry was a stale `buildkit-blob-1-sha256:…` layer, all
created 2026-10-18/19, and all attached to the two retired GHCR cutover branches:

| ref | entries |
|---|---|
| `refs/heads/feature/offline-ghcr-build` | 102 |
| `refs/heads/feature/ghcr-offline-build` | 66 |

**Why nothing would ever have cleaned it:** `cache-to` is gone from every
workflow (removed at the 10 GB quota under ROUTER-007). `deploy` and
`build-test` only *read* (`cache-from`). GitHub evicts cache only under write
pressure or idle expiry, so with no writer this was stranded indefinitely.

**Why purging is safe:** nothing writes the cache, so nothing depended on those
entries; a build repopulates whatever it needs. And they were near-useless
anyway — the most recent full deploy showed **exactly one** `CACHED` layer
across both image builds. Both consuming refs are retired cutover branches;
`master` builds read the default scope, which held none of this.

Purged via the REST API, one ID at a time (`DELETE /actions/caches` with no
`key`/`ref` returns 422 "Missing required query parameter key"). Ground truth
after: the cache listing returns **0 entries**. The `cache/usage` endpoint still
reported 165 entries at the time of writing — it lags; the listing is
authoritative.

## 3. Evaluated and rejected: switching the 9router Dockerfile to `npm ci`

Earlier notes (ROUTER-009) suggested `npm ci` as the fix for `build-test`'s
one-off `Module not found: Can't resolve '@aws-sdk/credential-providers'`
failure. **That is not a one-word change, and should not be done as a drive-by:**

- **`vianhanif/9router` ships no lockfile** — only `package.json`. `npm ci`
  cannot run without one. (`vianhanif/9router-api` does ship
  `package-lock.json`, and is not the repo that failed.)
- **The real fix would be generating and committing a first lockfile** in the
  app repo, which pins the whole transitive graph — including optional
  dependencies like `@aws-sdk/credential-providers` whose presence under
  `npm install` is opportunistic. That changes what is installed and therefore
  what is built.
- **The failure is transient and low-cost**: it occurred once, on a docs-only
  branch, and the next build of the same commit succeeded. `build-test` is a
  branch tester; a red run is noise. Production builds the pinned release, whose
  commit is already proven.

Revisit only if it recurs, and then as its own tracked task in the app repo.
