# syntax=docker/dockerfile:1.7
ARG NODE_IMAGE=node:22-alpine

# ---- Stage 1: fetch source from master ----
FROM ${NODE_IMAGE} AS source
ARG NINEROUTER_REPO=https://github.com/vianhanif/9router.git
ARG NINEROUTER_VERSION=master
RUN apk add --no-cache git
WORKDIR /app
# Fetch by ref (branch name or commit SHA). Using git init + fetch supports
# both, so CI can pass a resolved SHA to bust the clone-layer cache.
RUN git init /app/source && \
    cd /app/source && \
    git remote add origin ${NINEROUTER_REPO} && \
    git fetch --depth 1 origin ${NINEROUTER_VERSION} && \
    git checkout FETCH_HEAD

# ---- Stage 2: build ----
FROM ${NODE_IMAGE} AS builder
WORKDIR /app
RUN apk --no-cache upgrade && apk --no-cache add python3 make g++ linux-headers

COPY --from=source /app/source/package.json ./
RUN --mount=type=cache,target=/root/.npm npm install

COPY --from=source /app/source ./
ENV NEXT_TELEMETRY_DISABLED=1
RUN npm run build

# ---- Stage 3: runner ----
FROM ${NODE_IMAGE} AS runner
WORKDIR /app
LABEL org.opencontainers.image.title="9router"
ENV NODE_ENV=production \
    PORT=20128 \
    HOSTNAME=0.0.0.0 \
    NEXT_TELEMETRY_DISABLED=1 \
    DATA_DIR=/app/data

COPY --from=builder /app/public ./public
COPY --from=builder /app/.next/static ./.next/static
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/custom-server.js ./custom-server.js
COPY --from=builder /app/open-sse ./open-sse
# Next file tracing can omit sibling files; MITM runs server.js as a separate process.
COPY --from=builder /app/src/mitm ./src/mitm
# Standalone node_modules may omit deps only required by the MITM child process.
COPY --from=builder /app/node_modules/node-forge ./node_modules/node-forge
# node-machine-id is createRequire-loaded at runtime; tracing omits it.
COPY --from=builder /app/node_modules/node-machine-id ./node_modules/node-machine-id
# Ensure `next` is available at runtime in case tracing did not include it.
COPY --from=builder /app/node_modules/next ./node_modules/next
# sql.js loads dist/sql-wasm.wasm by path at runtime; tracing only follows JS imports,
# so the last-resort DB driver would abort with ENOENT on the missing binary.
COPY --from=builder /app/node_modules/sql.js ./node_modules/sql.js

RUN mkdir -p /app/data && chown -R node:node /app && \
  mkdir -p /app/data-home && chown node:node /app/data-home && \
  ln -sf /app/data-home /root/.9router 2>/dev/null || true

# Fix permissions at runtime (handles mounted volumes)
RUN apk --no-cache upgrade && apk --no-cache add su-exec && \
  printf '#!/bin/sh\nchown -R node:node /app/data /app/data-home 2>/dev/null\nexec su-exec node "$@"\n' > /entrypoint.sh && \
  chmod +x /entrypoint.sh

EXPOSE 20128

ENTRYPOINT ["/entrypoint.sh"]
CMD ["node", "custom-server.js"]