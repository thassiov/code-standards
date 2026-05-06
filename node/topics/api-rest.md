# Topic: REST API

> **Status:** stub. Will harden against a real API project.

## Recommended baseline

| Concern | Choice |
|---|---|
| Framework | Fastify (perf-leaning) or NestJS (opinionated, decorators) |
| Validation | Zod or Valibot for request schemas |
| OpenAPI | Generated from code: `@fastify/swagger` for Fastify, NestJS native |
| Logging | pino (Fastify default) |
| DB layer | drizzle / kysely for raw-ish SQL; sequelize for NestJS-shape projects |
| Auth | External service (Sump, Auth0, Clerk) or session-based via cookies |
| Rate limit | `@fastify/rate-limit` or `nestjs-throttler` |

## Patterns (to expand against a real project)

- Error handling: typed error classes mapped to HTTP statuses by a single error filter / handler. Don't `throw new Error('...')` from handlers — use specific subclasses.
- Validation at the boundary, no matter how trivial. Don't trust input.
- DTOs stay separate from domain entities. Map at the controller layer.
- Pagination: cursor-based for feeds, offset for admin lists. Document in OpenAPI.
- Idempotency keys for state-changing requests where retry is plausible.

## Cross-references

- Animaiz's NestJS shape (in `~/dev/personal/projects/animaiz/animaiz-backend/`) is the working reference.
- Gotchas around error handling, validation, and middleware order will be hoisted here as they accumulate.

## Open questions

- GraphQL standards (separate file when we have a real GraphQL project).
- WebSocket / SSE patterns.
- API versioning convention (`/v1/...` vs header-based).
