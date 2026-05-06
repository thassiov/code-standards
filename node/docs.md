# Documentation

README structure, JSDoc/TSDoc, ADRs.

## 1. README — minimum viable

Every repo has a `README.md` covering, in order:

1. **Title + one-line pitch** — what this project is in 12 words or less
2. **Status** — alpha / beta / stable / archived (one word)
3. **Quick start** — install + run the simplest possible example
4. **Usage** — typical use cases with code snippets
5. **Configuration** — env vars, config files, defaults
6. **Development** — clone, install, test, build (assume reader has Node + pnpm)
7. **License** — one line + link to LICENSE file

A one-page README is fine and often best. Long-form docs go in `docs/`
or per-package READMEs.

```markdown
# project-name

> One-line description of what this is.

**Status:** alpha · WIP · do not depend on this.

## Quick start

\`\`\`bash
pnpm install
pnpm dev
\`\`\`

Visit http://localhost:3000.

## Usage

\`\`\`ts
import { thing } from 'project-name';

await thing.doStuff({ ... });
\`\`\`

## Configuration

| Env var | Description | Default |
|---|---|---|
| `PORT` | HTTP port | `3000` |
| `LOG_LEVEL` | one of `debug`, `info`, `warn`, `error` | `info` |

## Development

\`\`\`bash
pnpm install
pnpm test
pnpm dev
\`\`\`

## License

MIT — see [LICENSE](LICENSE).
```

## 2. TSDoc / JSDoc

- **Every exported symbol gets a doc comment.** This is enforced by
  ESLint (`tsdoc/syntax` if you add the plugin, otherwise by review).
- TSDoc style (`/** ... */`) on TypeScript. JSDoc syntax (`@param`,
  `@returns`) is the same — the difference is mostly tooling.
- Don't restate the type in the comment; restate the *intent*.

```ts
/**
 * Resolves a Sump session cookie to an account ID.
 *
 * Returns null when the session is missing, expired, or revoked —
 * callers should treat all three the same and respond 401.
 *
 * @throws SumpUnreachableError when the auth service is down.
 */
export async function resolveAccount(cookie: string): Promise<string | null> {
  // ...
}
```

Rules:
- **Document the contract, not the mechanics.** "Returns null when X"
  beats "calls Sump's `/sessions/me` and parses the JSON response".
- **Document `@throws`** for every error type a caller might want to
  catch. Don't list internal errors that bubble up unchanged.
- **Don't write `@param description for paramName`** when the type and
  name already say everything. `@param cookie - the session cookie`
  is noise.
- **One-line summary on its own line.** Longer body separated by a
  blank line. Tools render the summary as a tooltip.

## 3. ADRs (Architecture Decision Records)

For non-trivial decisions, write an ADR. They live in `docs/decisions/`
or `docs/adr/`, numbered sequentially.

`docs/decisions/0001-postgres-over-mongo.md`:

```markdown
# 1. PostgreSQL over MongoDB

Date: 2026-05-04

## Status

Accepted

## Context

We need a primary datastore for [project]. Two main candidates were
considered: PostgreSQL and MongoDB. Both have been used in past
iterations of this project family.

## Decision

We will use PostgreSQL.

## Rationale

- Relational shape is a better fit for the dual-nature Listing model.
- MongoDB's schemaless flexibility was an early-stage benefit but
  produced inconsistencies in past iterations.
- Postgres' JSONB columns cover the cases we'd reach for Mongo for.
- Existing infrastructure runs Postgres for other projects.

## Consequences

- We commit to writing migrations (drizzle / kysely / raw SQL — TBD).
- ORM choice becomes a follow-up decision.
- Geospatial queries will use PostGIS, not Mongo's geospatial features.
```

Rules:
- **One decision per ADR.** Don't bundle.
- **Status is one of `Proposed` / `Accepted` / `Deprecated` / `Superseded by [N]`.**
- **Don't edit accepted ADRs.** If a decision changes, write a new ADR
  and mark the old one `Superseded by [N]`. ADRs are append-only.
- **Date them.** It's how readers know if the ADR is still current.

## 4. Inline comments

- **The code says *what*, the comment says *why*.** If you have to
  explain *what*, the code is unclear.
- Workarounds, hacks, and surprising decisions get a comment with a
  ticket / ADR reference if relevant.
- **TODOs include an owner or context**: `// TODO(thassiov): ...` or
  `// TODO(animaiz#42): ...`. Bare `// TODO` is an alarm clock with no
  alarm set.
- **No commented-out code.** Git remembers. Delete it.

## 5. Generated docs

- TypeScript projects that are **published as libraries** — generate
  reference docs via TypeDoc into `docs/api/`. CI publishes to GH Pages.
- TypeScript projects that are **applications** — don't bother with
  TypeDoc. The README + ADRs + inline TSDoc are enough.
- HTTP APIs — generate OpenAPI from code (NestJS does this natively;
  for raw Fastify, use `@fastify/swagger`). Commit the generated YAML
  to `docs/api/openapi.yaml` so consumers can diff it.

## 6. Cross-references in docs

- Link to other docs in the same repo with relative paths
  (`[link](../other.md)`). Don't link to the GitHub URL — it breaks
  on forks and mirrors.
- Link to source files with relative paths too: `[`src/foo.ts`](../src/foo.ts)`.
  Line-anchor links (`#L12`) are fine but rot fast.
