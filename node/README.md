# Node / JavaScript / TypeScript Project Standards

Self-contained standards for personal Node.js projects: CLIs, HTTP APIs,
browser extensions, and the small-to-medium services that connect them.
The scope companion to [`go.md`](../go.md).

**Scope:** TypeScript-first projects on Node 22+. Pure JavaScript is
allowed but should adopt JSDoc-typed style; long-term goal is to migrate
JS-only repos to TS.

**How to use this dir:**
1. Read `general.md` and `tooling.md` for the mental model.
2. For a new project, work through the checklist in `checklist.md`,
   copying templates from the relevant section files.
3. Topical files (`topics/*.md`) layer on top of the foundation when the
   project type calls for them — pick only what applies.
4. `gotchas.md` is the pre-flight checklist for things we've been burned
   by.

## Files

### Foundation (always relevant)

| File | Covers |
|---|---|
| [general.md](general.md) | Repo layout, language version, package manager, naming, file organization |
| [tooling.md](tooling.md) | TypeScript config, ESLint, Prettier, package scripts |
| [tests.md](tests.md) | Test framework choice, structure, naming, fixtures |
| [docs.md](docs.md) | README structure, JSDoc/TSDoc, ADRs |
| [ci.md](ci.md) | GitHub Actions workflow templates |
| [checklist.md](checklist.md) | Step-by-step new-project checklist |
| [gotchas.md](gotchas.md) | Stuff we've been burned by |

### Topics (use only what applies)

| File | When to use |
|---|---|
| [topics/cli.md](topics/cli.md) | Single-binary CLI tools |
| [topics/api-rest.md](topics/api-rest.md) | REST/HTTP APIs (Express/Fastify/Hono) |
| [topics/aws.md](topics/aws.md) | AWS SDK v3, Lambda, env-driven config |
| [topics/containers.md](topics/containers.md) | Dockerfiles for Node services |
| [topics/browser-extension.md](topics/browser-extension.md) | WebExtension MV3 (Chrome/Firefox/Edge) |

## Status

This directory is **scaffolding** as of 2026-05-04. Foundation files are
opinionated and ready; topical files start as stubs and harden as
projects exercise them. Do not treat any file as complete until it's
been [tested](../testing-standards.md) against a scaffolded project.

## What's intentionally NOT here (yet)

- GraphQL APIs (no canonical pattern; pick per-project, document in repo)
- Frontend framework standards (React/Next.js conventions live in the
  individual project repos for now)
- Database migration tooling (drizzle / prisma / kysely / raw SQL — pick
  per-project)
- Monorepo orchestration (pnpm workspaces, turbo, nx) — pattern still
  shaking out
- Observability (metrics/tracing/logs aggregation) — same status as `go.md` §17
