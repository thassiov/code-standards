# Tests

Test framework, structure, naming, fixtures.

## 1. Framework choice

- **Vitest** is the default for new projects. Native TS, fast, near-Jest API.
- **Jest** is acceptable for projects that already use it (e.g. NestJS
  apps where Jest is the framework default). Don't migrate without reason.
- **Playwright** for browser E2E. No alternative considered.
- **Supertest** with whichever runner for HTTP-handler integration tests.

## 2. Co-located tests

Test files live next to the code they test, named `<source>.test.ts`:

```
src/
├── feed/
│   ├── feed.ts
│   ├── feed.test.ts
│   └── ranker.ts
└── auth/
    ├── auth.ts
    └── auth.test.ts
```

Integration / E2E tests that need fixtures or real services live in a
sibling directory:

```
test/
├── integration/
│   ├── api.test.ts
│   └── fixtures/
└── e2e/
    └── playwright/
```

## 3. Vitest config

`vitest.config.ts`:

```ts
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    globals: false, // import { describe, it, expect } explicitly
    environment: 'node', // 'happy-dom' for DOM tests, 'jsdom' if you must
    coverage: {
      provider: 'v8',
      reporter: ['text', 'lcov', 'html'],
      include: ['src/**/*.ts'],
      exclude: ['src/**/*.test.ts', 'src/**/index.ts', 'src/**/*.d.ts'],
      thresholds: {
        lines: 80,
        functions: 80,
        branches: 75,
        statements: 80,
      },
    },
  },
});
```

For browser-extension projects, use `'happy-dom'` (lighter than jsdom).

## 4. Test structure — what good looks like

### Unit test — table-driven

```ts
import { describe, it, expect } from 'vitest';
import { parseDuration } from './duration.js';

describe('parseDuration', () => {
  const cases: Array<{ name: string; input: string; want: number | null }> = [
    { name: 'plain seconds', input: '30s', want: 30_000 },
    { name: 'minutes',       input: '5m',  want: 300_000 },
    { name: 'hours',         input: '2h',  want: 7_200_000 },
    { name: 'invalid',       input: 'xyz', want: null },
    { name: 'empty',         input: '',    want: null },
  ];

  for (const c of cases) {
    it(c.name, () => {
      expect(parseDuration(c.input)).toBe(c.want);
    });
  }
});
```

### Integration test — real HTTP

```ts
import { afterAll, beforeAll, describe, expect, it } from 'vitest';
import request from 'supertest';
import { createApp } from '../src/app.js';

describe('GET /listings', () => {
  let app: ReturnType<typeof createApp>;

  beforeAll(() => {
    app = createApp({ port: 0 });
  });

  afterAll(async () => {
    await app.close();
  });

  it('returns 200 with empty list on empty db', async () => {
    const res = await request(app.server).get('/listings');
    expect(res.status).toBe(200);
    expect(res.body).toEqual({ items: [], total: 0 });
  });
});
```

## 5. Rules

- **`describe` blocks describe the *unit*, `it` blocks describe the *behavior*.**
  Read out loud: `describe parseDuration > it parses plain seconds` ≈ "parseDuration parses plain seconds." Good.
- **One assertion per test in the happy path.** Multiple is fine if
  they're all aspects of the same outcome (status code + body + header).
- **Arrange / Act / Assert with blank lines** between phases. Easier to
  scan than tightly-packed tests.
- **No shared mutable state between tests.** Each test sets up what it needs.
  `beforeEach` is fine for re-creating fixtures; `beforeAll` only for
  truly read-only setup (loading a config, starting a server).
- **Mock at the module boundary**, not by stubbing methods. Use
  `vi.mock('./api')` to swap a whole module. For HTTP, use a fake server
  (e.g. `msw`) instead of stubbing `fetch`.
- **Async is `await`, not callbacks.** `done` callbacks are banned.
- **Test names are sentences**, not method-name echoes:
  - ❌ `it('parseDuration_returnsNullForEmpty')`
  - ✅ `it('returns null for empty input')`
- **`test.skip` and `it.only` never get committed.** Lint rule:
  `vitest/no-focused-tests` and `vitest/no-disabled-tests`.

## 6. Coverage philosophy

- 80% line coverage as a default floor. Higher for libraries, lower for
  one-off scripts.
- **Don't write tests just to hit coverage.** Untested code that's
  trivial-by-inspection (DTO mappers, glue) is fine. Track the *uncovered
  branches* — those are usually the bug-magnets.
- Branch coverage matters more than line coverage. A single line with
  `cond ? a : b` needs two tests, not one.
- Fail CI below threshold. The threshold lives in `vitest.config.ts`,
  not as a CLI flag, so it's enforced uniformly.

## 7. Fixtures and factories

- Build factories, not fixtures. A factory function returns a sensible
  object that callers override per-test:

  ```ts
  export function makeListing(overrides: Partial<Listing> = {}): Listing {
    return {
      id: crypto.randomUUID(),
      type: 'adoption',
      status: 'active',
      title: 'Test listing',
      content: '',
      authorProfileId: crypto.randomUUID(),
      createdAt: new Date(),
      ...overrides,
    };
  }
  ```

- **Don't import factory libraries** (faker, factory-girl-style) unless
  the project genuinely needs randomized data. Per-test `makeFoo()`
  helpers are clearer.

- Fixtures with serialized JSON / HTTP responses live under
  `test/fixtures/` or `__fixtures__/`. Load via `readFileSync` —
  `import` of `.json` works but loses type safety.

## 8. Test database / external services

- **Integration tests get their own database**, not the dev one. Either
  a fresh in-memory SQLite, or a containerized Postgres started by the
  test setup. Never share state with the dev DB.
- **Don't mock the database.** Mock the boundary above it (the repository
  / data-access layer) for unit tests, OR run against a real DB for
  integration tests. Mock-DB-with-fake-rows is the worst of both worlds.
- For Postgres in tests: `testcontainers` (Node lib) is the cleanest
  option. Spins up a real Postgres in Docker per suite.
- For SQLite in tests: just open `:memory:` or a temp file per suite.

## 9. E2E tests (Playwright)

- Live in `test/e2e/` or `e2e/`. Don't co-locate with source.
- One `<flow>.spec.ts` per user flow, not per page.
- Use page-object pattern only when E2E suite grows past ~20 specs;
  earlier than that, helper functions in `test/e2e/helpers/` are simpler.
- CI runs E2E only on a "real-ish" environment (Docker compose stack
  or staging). Don't try to run Playwright against mocked backends.
- Visual regression is opt-in and per-spec — `await
  expect(page).toHaveScreenshot()`.

## 10. Cross-reference

The meta-test for the *standards doc itself* lives in
[`testing-standards.md`](../testing-standards.md). When adding new
patterns to this file, dogfood them by scaffolding a tiny project and
running through.
