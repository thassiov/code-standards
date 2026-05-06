# Topic: Containers

> **Status:** stub. Will harden against a real containerized service.

## Recommended Dockerfile baseline

Multi-stage, distroless-ish runtime, non-root user. Adjust per project.

```dockerfile
# syntax=docker/dockerfile:1.7

# ---- deps ----
FROM node:22-alpine AS deps
WORKDIR /app
RUN corepack enable
COPY package.json pnpm-lock.yaml ./
RUN --mount=type=cache,id=pnpm,target=/root/.local/share/pnpm/store \
    pnpm install --frozen-lockfile --prod=false

# ---- build ----
FROM deps AS build
COPY . .
RUN pnpm build && pnpm prune --prod

# ---- runtime ----
FROM node:22-alpine AS runtime
WORKDIR /app
ENV NODE_ENV=production
RUN addgroup -S app && adduser -S app -G app
COPY --from=build --chown=app:app /app/node_modules ./node_modules
COPY --from=build --chown=app:app /app/dist ./dist
COPY --from=build --chown=app:app /app/package.json ./package.json
USER app
EXPOSE 3000
CMD ["node", "dist/index.js"]
```

## Rules

- **Multi-stage.** Build stage has dev deps, runtime stage doesn't.
- **Non-root user.** Always.
- **Pin the base image.** `node:22-alpine` rolls; pin to a digest
  (`node:22-alpine@sha256:...`) for reproducibility on long-lived
  services.
- **`COPY --link` and `--mount=type=cache`** for Docker BuildKit
  speed-ups when the project has many deps.
- **`HEALTHCHECK`** for HTTP services. The orchestrator (compose,
  Kubernetes, ECS) usually does its own, but `HEALTHCHECK` makes
  `docker ps` show health.
- **No secrets in images.** Pass at runtime via env / mounted files.
- **`.dockerignore`** is as important as the Dockerfile. Without it
  you copy `node_modules`, `.git`, `dist` into the build context and
  cache invalidates constantly.

## `.dockerignore` template

```
node_modules
dist
.git
.github
coverage
*.log
.env
.env.*
README.md
```

## Compose

`docker-compose.yaml` is the standard local-dev orchestrator. Treat it
as documentation for the runtime topology, not just a dev convenience.

## Open questions

- Whether to standardize Alpine vs Debian-slim. Alpine smaller, but
  `musl` vs `glibc` compatibility surprises happen.
- Best pattern for monorepo containerization (build all packages once,
  ship N images? Or one image per service with `pnpm deploy`?).
