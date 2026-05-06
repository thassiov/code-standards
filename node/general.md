# General

Language baseline, repo layout, package manager, naming.

## 1. Language version

- **TypeScript 5.x** is the default. Pure JavaScript is permitted but
  must adopt JSDoc-typed style (`// @ts-check` + JSDoc annotations) and
  is on the migration path to TS.
- **Node 22 LTS** is the runtime target. Pin via `.nvmrc` and the
  `engines.node` field in `package.json`. CI uses the same version.
- ECMAScript module (`"type": "module"`) by default. CJS only for CLIs
  that need to interoperate with old toolchains; document the choice in
  the project README.

## 2. Package manager

- **pnpm** is the default. Faster, supports workspaces natively, no
  hoisting surprises.
- Lockfile (`pnpm-lock.yaml`) is committed. Always.
- For repos already on `npm` or `yarn`, don't migrate without a reason —
  but new repos start on pnpm.

## 3. Repo layout

### Single-package project

```
<project>/
├── src/
│   ├── index.ts             # entry point (CLI / lib export / server bootstrap)
│   └── ...                  # one dir per bounded concern
├── test/                    # only if tests don't live next to source
├── scripts/                 # one-shot dev scripts
├── docs/                    # ADRs, deeper docs (only if README outgrows itself)
├── .github/workflows/
│   └── ci.yml
├── .nvmrc                   # node version, e.g. "22"
├── .gitignore
├── eslint.config.js         # flat config
├── .prettierrc
├── tsconfig.json
├── package.json
├── pnpm-lock.yaml
├── README.md
└── LICENSE
```

### Monorepo (pnpm workspaces)

```
<project>/
├── packages/
│   ├── core/                # shared types/utils
│   ├── api/                 # backend service
│   ├── cli/                 # CLI binary
│   └── web/                 # frontend
├── pnpm-workspace.yaml
├── package.json             # root: shared scripts, devDependencies
└── ...                      # rest as above, but per-package configs are inherited where possible
```

Rules:
- **`src/` is the default home for new code.** Tests live alongside as
  `*.test.ts` (Vitest/Jest pattern), not in a separate `test/` dir,
  unless integration tests need it.
- **One bounded concern per top-level `src/<dir>/`.** Avoid dumping-ground
  names: `utils`, `helpers`, `common`, `lib`. Be specific:
  `src/storage/`, `src/auth/`, `src/feed/`.
- **Don't nest more than three levels deep** under `src/`. Three is the
  smell threshold; collapse to siblings.
- **`scripts/` is for repo-local dev/ops scripts** (db reset, fixture
  generators). Not the application's own commands.

## 4. File and identifier naming

| What | Convention |
|---|---|
| Files (TS/JS source) | `kebab-case.ts` |
| Files (configs, READMEs) | `kebab-case` or `PascalCase` per ecosystem (`README.md`, `Dockerfile`) |
| Directories | `kebab-case` |
| Types/Interfaces/Classes | `PascalCase` |
| Variables/Functions | `camelCase` |
| Constants (truly fixed) | `SCREAMING_SNAKE_CASE` |
| Enums | `PascalCase` for enum, `PascalCase` for members |
| React components (when applicable) | `PascalCase.tsx` |
| Test files | `<source>.test.ts` |

Avoid `index.ts` re-export barrels deeper than one level — they hurt
tree-shaking and add layers of indirection. Acceptable at the root of
a `src/<module>/` if it really has a public surface.

## 5. TypeScript discipline

- `strict: true` always. No exceptions.
- `noUncheckedIndexedAccess: true` — array/record access returns `T | undefined`.
- `exactOptionalPropertyTypes: true` — distinguish `{ x?: T }` from `{ x: T | undefined }`.
- `noImplicitOverride: true` — explicit `override` keyword in subclasses.
- **No `any`.** Use `unknown` and narrow. If you must use `any`, add a
  `// eslint-disable-next-line @typescript-eslint/no-explicit-any` with
  a comment explaining why.
- **No type assertions (`as Foo`)** unless the value genuinely came from
  outside the type system (parsed JSON, FFI). Prefer type guards.
- **`as const` is fine** — it's a contract, not an escape hatch.

See [`tooling.md`](tooling.md) for the full `tsconfig.json` template.

## 6. Module imports

- Use ES module syntax (`import x from 'y'`).
- Path aliases (`@/foo/bar`) only in apps with bundlers that resolve
  them. Backend Node code uses relative paths (`./foo/bar`) — keeps
  `tsx`/`node --import` working without extra hooks.
- Group import order: built-in → external → internal-absolute →
  internal-relative. Enforced by ESLint (`import-x/order`).
- Side-effect imports (`import './polyfills'`) go at the top, separated
  by a blank line.

## 7. Errors and assertions

- **Throw `Error` subclasses.** Never throw strings.
- One named subclass per failure mode that callers branch on:
  ```ts
  export class NotFoundError extends Error {
    constructor(public readonly resource: string) {
      super(`${resource} not found`);
      this.name = 'NotFoundError';
    }
  }
  ```
- **Don't use `assert` from `node:assert`** in production code — it
  throws `AssertionError` which leaks "internal" wording. Use guard
  clauses with explicit error types instead.
- **Always `await` promises that can reject** or chain `.catch`. Floating
  promises are a lint error.

## 8. Async patterns

- `async`/`await` everywhere; no `.then().then()` chains in new code.
- **Wrap `Promise.all` calls when individual failures matter** —
  `Promise.allSettled` if you need to inspect both successes and failures.
- Never `await` inside a `forEach` — it's a no-op silently. Use `for...of`
  with `await`, or `Promise.all(arr.map(async ...))`.
- Cancellation via `AbortSignal` — pass through APIs, never construct a
  fresh `AbortController` inside a function unless the function genuinely
  owns that lifecycle.

## 9. What goes in `package.json`

- `name`, `version`, `description`, `license` always.
- `engines.node` matches `.nvmrc`.
- `type: "module"` (or `"commonjs"` if you have a reason).
- `main`, `types`, `exports` if the package is consumed by others.
- `bin` only if the package ships a CLI.
- `scripts` follow conventions: `build`, `dev`, `test`, `test:watch`,
  `lint`, `lint:fix`, `format`, `format:check`, `typecheck`, `ci`.
- `dependencies` vs `devDependencies` — get this right; tools that scan
  for vulns separate the two.
- **Don't put scripts in `package.json` that just call other scripts in
  the same file** ("script call chains"). Just inline the command. Use
  shell composition (`&&`) when chaining.
