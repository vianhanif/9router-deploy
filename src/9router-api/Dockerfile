# syntax=docker/dockerfile:1

# ---- Stage 1: 9router dependency tree ----
FROM node:22-alpine AS ninesrc
ARG NINEROUTER_REPO=https://github.com/vianhanif/9router.git
ARG NINEROUTER_VERSION=master
RUN apk add --no-cache git python3 make g++ linux-headers
WORKDIR /app
# Fetch by ref (branch name or commit SHA).
RUN git init /app/9router && \
    cd /app/9router && \
    git remote add origin ${NINEROUTER_REPO} && \
    git fetch --depth 1 origin ${NINEROUTER_VERSION} && \
    git checkout FETCH_HEAD
WORKDIR /app/9router
# Fix arborist crash in npm 10.9.8
RUN npm install -g npm@11 && npm install --omit=dev --no-audit --no-fund

# ---- Stage 2: 9router-api build ----
FROM node:22-alpine AS apibuild
ARG APIREPO_REPO=https://github.com/vianhanif/9router-api.git
ARG APIREPO_VERSION=master
RUN apk add --no-cache git python3 make g++ linux-headers
WORKDIR /app
# Fetch by ref (branch name or commit SHA).
RUN git init /app/9router-api && \
    cd /app/9router-api && \
    git remote add origin ${APIREPO_REPO} && \
    git fetch --depth 1 origin ${APIREPO_VERSION} && \
    git checkout FETCH_HEAD
WORKDIR /app/9router-api
# Make 9router source available as a sibling so tsconfig path aliases
# (@/* -> ../9router/src/*, open-sse/* -> ../9router/open-sse/*) resolve
# during the esbuild bundle. Mirrors Stage 3, which copies /app/9router too.
COPY --from=ninesrc /app/9router /app/9router
# Fix arborist crash in npm 10.9.8
RUN npm install -g npm@11 && npm install --no-audit --no-fund
RUN npm run build   # esbuild -> dist/server.js

# ---- Stage 2.5: headroom-ai Python venv ----
# Build toolchain + pip live ONLY here; the runner copies just the venv,
# keeping it free of gcc/musl-dev/headers. Same base tag as the runner so
# the python3 version and libc match. Debian (glibc) required: headroom-ai
# >=0.21 ships only manylinux_2_28 wheels, no musllinux (Alpine).
FROM node:22-bookworm-slim AS pyheadroom
RUN apt-get update && apt-get install -y --no-install-recommends python3 python3-venv \
    && rm -rf /var/lib/apt/lists/*
RUN python3 -m venv /opt/headroom-venv \
    && /opt/headroom-venv/bin/pip install --no-cache-dir --only-binary :all: "headroom-ai[proxy,code]==0.37.0"
RUN find /opt/headroom-venv -name __pycache__ -type d -prune -exec rm -rf {} + \
    && rm -rf /root/.cache/pip

# ---- Stage 3: runner ----
FROM node:22-bookworm-slim
ENV NODE_ENV=production \
    PORT=20127 \
    NINEROUTER_HOME=/app/9router \
    HEADROOM_PYTHON=/opt/headroom-venv/bin/python3
RUN apt-get update && apt-get install -y --no-install-recommends python3 \
    && rm -rf /var/lib/apt/lists/* \
    && mkdir -p /app/9router /app/data /app/data-home \
    && printf '#!/bin/sh\nchown -R node:node /app/data /app/data-home 2>/dev/null\nchown -R node:node /app/9router-api /app/9router 2>/dev/null\nexec setpriv --reuid=node --regid=node --init-groups "$@"\n' > /entrypoint.sh \
    && chmod +x /entrypoint.sh \
    && ln -s /app/data-home /root/.9router

COPY --from=ninesrc /app/9router /app/9router
COPY --from=apibuild /app/9router-api/node_modules /app/9router-api/node_modules
COPY --from=apibuild /app/9router-api/dist /app/9router-api/dist
COPY --from=apibuild /app/9router-api/package.json /app/9router-api/package.json
COPY --from=pyheadroom /opt/headroom-venv /opt/headroom-venv
# server.ts does `require('./package.json')` at runtime; bundled dist/server.js
# resolves that relative to dist/, so mirror package.json there too.
RUN cp /app/9router-api/package.json /app/9router-api/dist/package.json
# CRITICAL: symlink so bare '9router/*' imports resolve at runtime
RUN ln -s /app/9router /app/9router-api/node_modules/9router
# Runtime `@/*` alias resolution via Node ESM loader. 9router source (kept
# external in the esbuild bundle) uses `@/x` and `open-sse/x` as bare
# specifiers — these are Next.js / tsconfig aliases, not real packages. This
# loader rewrites them to real file URLs under /app/9router/src and
# /app/9router/open-sse at import time. Baked into the image via printf so no
# repo source file is needed.
RUN cat > /app/9router-api/alias-loader.mjs <<'EOF'
import { register } from 'node:module';
import { pathToFileURL } from 'node:url';
register(new URL('./alias-hook.mjs', import.meta.url), pathToFileURL('/'));
EOF
RUN cat > /app/9router-api/alias-hook.mjs <<'EOF'
import { pathToFileURL, fileURLToPath } from 'node:url';
import { dirname, join } from 'node:path';
import { existsSync, statSync } from 'node:fs';

const ROOT = '/app/9router';

// Resolve a possibly-extensionless base path to a real file, trying the
// explicit path, then .js/.mjs/.cjs, then dir/index.* (mirrors Next/turbopack).
function lookup(base) {
  const candidates = [base, base + '.js', base + '.mjs', base + '.cjs'];
  for (const c of candidates) {
    if (existsSync(c) && statSync(c).isFile()) return c;
  }
  for (const idx of ['/index.js', '/index.mjs', '/index.cjs']) {
    const c = base + idx;
    if (existsSync(c) && statSync(c).isFile()) return c;
  }
  return null;
}

export async function resolve(specifier, context, nextResolve) {
  let base = null;
  if (specifier.startsWith('@/')) {
    base = ROOT + '/src/' + specifier.slice(2);
  } else if (specifier.startsWith('open-sse/')) {
    base = ROOT + '/open-sse/' + specifier.slice(9);
  } else if ((specifier.startsWith('./') || specifier.startsWith('../')) &&
             context.parentURL && context.parentURL.startsWith('file://' + ROOT)) {
    base = join(dirname(fileURLToPath(context.parentURL)), specifier);
  }
  if (base) {
    const hit = lookup(base);
    if (hit) {
      return { url: pathToFileURL(hit).href, shortCircuit: true };
    }
  }
  return nextResolve(specifier, context);
}
EOF
WORKDIR /app/9router-api
EXPOSE 20127
ENTRYPOINT ["/entrypoint.sh"]
CMD ["node", "--import", "/app/9router-api/alias-loader.mjs", "dist/server.js"]