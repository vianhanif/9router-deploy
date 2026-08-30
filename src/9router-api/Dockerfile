# syntax=docker/dockerfile:1

# ---- Stage 1: 9router dependency tree ----
FROM node:22-alpine AS ninesrc
ARG NINEROUTER_REPO=https://github.com/vianhanif/9router.git
ARG NINEROUTER_VERSION=master
RUN apk add --no-cache git python3 make g++ linux-headers
WORKDIR /app
RUN git clone --depth 1 --branch ${NINEROUTER_VERSION} ${NINEROUTER_REPO} /app/9router
WORKDIR /app/9router
RUN npm install --omit=dev --no-audit --no-fund

# ---- Stage 2: 9router-api build ----
FROM node:22-alpine AS apibuild
ARG APIREPO_REPO=https://github.com/vianhanif/9router-api.git
ARG APIREPO_VERSION=master
RUN apk add --no-cache git python3 make g++ linux-headers
WORKDIR /app
RUN git clone --depth 1 --branch ${APIREPO_VERSION} ${APIREPO_REPO} /app/9router-api
WORKDIR /app/9router-api
RUN npm install --no-audit --no-fund
RUN npm run build   # esbuild -> dist/server.js

# ---- Stage 3: runner ----
FROM node:22-alpine
ENV NODE_ENV=production \
    PORT=20127 \
    NINEROUTER_HOME=/app/9router
RUN apk add --no-cache su-exec \
    && mkdir -p /app/9router /app/data /app/data-home \
    && printf '#!/bin/sh\nchown -R node:node /app/data /app/data-home 2>/dev/null\nchown -R node:node /app/9router-api /app/9router 2>/dev/null\nexec su-exec node "$@"\n' > /entrypoint.sh \
    && chmod +x /entrypoint.sh \
    && ln -s /app/data-home /root/.9router

COPY --from=ninesrc /app/9router /app/9router
COPY --from=apibuild /app/9router-api/node_modules /app/9router-api/node_modules
COPY --from=apibuild /app/9router-api/dist /app/9router-api/dist
COPY --from=apibuild /app/9router-api/package.json /app/9router-api/package.json

# Runtime alias loader: dist/server.js keeps bare `@/...` and `open-sse/...`
# imports (esbuild --external:@/*). These hooks re-map them to the 9router repo
# at runtime (same mapping as tsconfig paths: @/* -> ../9router/src/*).
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

# CRITICAL: symlink so bare '9router/*' imports resolve at runtime
RUN ln -s /app/9router /app/9router-api/node_modules/9router
WORKDIR /app/9router-api
EXPOSE 20127
ENTRYPOINT ["/entrypoint.sh"]
CMD ["node", "--import", "/app/9router-api/alias-loader.mjs", "dist/server.js"]