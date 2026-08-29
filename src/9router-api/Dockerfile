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
RUN apk add --no-cache python3 make g++ linux-headers
WORKDIR /app/9router-api
COPY package.json package-lock.json ./
COPY server.ts tsconfig.json ./
COPY src/ ./src/
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
# CRITICAL: symlink so bare '9router/*' imports resolve at runtime
RUN ln -s /app/9router /app/9router-api/node_modules/9router
WORKDIR /app/9router-api
EXPOSE 20127
ENTRYPOINT ["/entrypoint.sh"]
CMD ["node", "dist/server.js"]