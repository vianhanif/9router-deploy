# AGENTS.md for 9router-deploy

## Purpose
This repository (`9router-deploy`) manages the deployment of the 9router dashboard and API on Tencent Cloud. 

## Key Policies
- **Security:** Do not commit `.env` files or secret keys (e.g., `cloudflared` JSON credentials, `TENCENT_SSH_KEY`).
- **Builds:** 9router and 9router-api are built from source at deploy time. Do not modify Dockerfiles without testing smoke builds first.
- **Proxy/Tunnel:** Caddy and Cloudflare Tunnel must remain functional; do not disrupt proxy config (`proxy/Caddyfile`).

## Workflow
1. **Planning:** Review deployment docs/CI workflows before any change.
2. **Coding:** Keep changes scoped to deployment infrastructure; keep apps in `src/`.
3. **Validation:** Smoke test new images with the `docker run` pattern in `.github/workflows/deploy.yml` before merging to master.
4. **Ops:** SSH to the VPS (`ssh tencent-cloud`) to verify health (`docker compose ps`).

## CI/CD
- `master` branch triggers deployments to Tencent Cloud.
- CI ensures build/smoke passes before updating the production stack.
