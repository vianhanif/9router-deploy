# AGENTS.md for 9router-deploy

## Purpose
This repository (`9router-deploy`) manages the deployment of the 9router dashboard and API on Tencent Cloud. 

## Key Policies
- **Security:** Do not commit `.env` files or secret keys (e.g., `cloudflared` JSON credentials, `TENCENT_SSH_KEY`).
- **Builds:** 9router and 9router-api are built on GitHub-hosted runners (not on the VPS) from source in the upstream repos (`github.com/vianhanif/9router`, `github.com/vianhanif/9router-api`). The VPS only pulls prebuilt images from GHCR. Images are tagged with the commit SHA for rollback safety — a production rollback uses the local `:prev` tag on the VPS, not the registry. Do not modify the Dockerfiles in `src/` without testing — they control the build environment. Edits to `src/9router/` or `src/9router-api/` (which only contain Dockerfiles, not app source) are silently ignored; edit the upstream repos instead.
- **Proxy/Tunnel:** Caddy and Cloudflare Tunnel must remain functional; do not disrupt proxy config (`proxy/Caddyfile`).

## Workflow
1. **Planning:** Review deployment docs/CI workflows before any change.
2. **Coding:** Keep changes scoped to deployment infrastructure; apps stay in upstream repos (this repo only holds Dockerfiles, not app source).
3. **Validation:** The `deploy.yml` workflow runs a smoke test on the VPS before touching production — it pulls the SHA-tagged images from GHCR, boots them, and verifies `/api/health` + route presence. Smoke test **must pass** before the prod switch (it's a hard gate; failure triggers rollback). Do NOT edit `src/` app code here — it's ignored; edit the upstream repos instead.
4. **Ops:** SSH to the VPS (`ssh tencent-cloud`) to verify health (`docker compose ps`).

## CI/CD
- `master` branch triggers deployments to Tencent Cloud.
- CI ensures build/smoke passes before updating the production stack.
