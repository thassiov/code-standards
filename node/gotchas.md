# Gotchas

Stuff we've been burned by. Read before shipping.

## TypeScript

- **`strict: true` doesn't include `noUncheckedIndexedAccess`.** Without it,
  `arr[0]` is typed as `T`, not `T | undefined`. Always enable explicitly.
- **`exactOptionalPropertyTypes` and React props don't always play nice** —
  some libraries' types assume `{ x?: T }` accepts `undefined`. Workaround:
  type-cast at the boundary or use `Partial`.
- **`verbatimModuleSyntax: true`** breaks if you have any `import x from 'y'`
  where `x` is type-only and `y` has runtime side effects. The fix
  (`import type { x }`) is correct, but the migration error message is unhelpful.
- **`tsc --noEmit` is the slowest part of CI.** For monorepos, project
  references (`tsconfig.json` with `"references": [...]`) materially help.
- **`isolatedModules: true` requires `import type` for type-only imports.**
  Catches you the first time it bites; second time it never does.

## ESM / CJS

- **`"type": "module"` requires `.js` extensions in imports**, even from
  `.ts` files. `import './foo'` fails at runtime; `import './foo.js'`
  works (TypeScript resolves it correctly via `moduleResolution: NodeNext`).
- **`require` doesn't exist in ESM.** Use `import` or
  `createRequire(import.meta.url)` if you must interop.
- **`__dirname` doesn't exist in ESM.** Use
  `path.dirname(fileURLToPath(import.meta.url))` or `import.meta.dirname`
  (Node 20.11+).
- **A package can be ESM-only and break your CJS app.** `chalk@5`,
  `node-fetch@3`, etc. Either upgrade your project to ESM or pin the
  dependency to its last CJS version.

## pnpm

- **`pnpm install --frozen-lockfile` fails if `package.json` and lockfile
  disagree.** It's *supposed* to. Don't `--no-frozen-lockfile` in CI to
  paper over a real lockfile/manifest mismatch.
- **pnpm hoists differently from npm.** Code that worked under npm because
  of accidental hoisting will fail under pnpm with `Cannot find module`.
  Add the missing dep explicitly; don't use `shamefully-hoist`.
- **Workspaces and `workspace:` protocol** — `"foo": "workspace:*"` in a
  monorepo. When publishing, pnpm rewrites to a real version. Don't put
  `workspace:*` in a published package; use `workspace:^` for dependents.

## Vitest / Jest

- **Vitest auto-imports `globals` if `globals: true` in config.** That
  hides `describe`/`it`/`expect` imports — but breaks IDE go-to-definition.
  Set `globals: false` and import explicitly.
- **`vi.mock` is hoisted.** Calls to it run before any other top-level
  code in the test file. Variables referenced inside the factory must
  also be hoisted (use `vi.hoisted(...)` for setup state).
- **Async test that doesn't `await`** silently passes (the assertion
  promise rejects out of band). ESLint `vitest/expect-expect` catches the
  no-await case. Always `await expect(...)` when matchers return promises.

## ESLint

- **`@typescript-eslint/strict-type-checked` requires
  `parserOptions.project`.** Slow on big repos. Two paths:
  1. Use `projectService: true` (newer, faster, beta).
  2. Sub-tsconfigs scoped to the source dirs you actually lint.
- **The flat config (`eslint.config.js`) doesn't read `.eslintrc*`.**
  Migrating from old config is mostly mechanical but watch for plugins
  that haven't shipped flat-config support yet.
- **`eslint --fix` can produce non-idempotent fixes.** Run it twice
  before assuming it's done.

## Prettier

- **Prettier and ESLint stylistic rules will fight** if both are enabled.
  `eslint-config-prettier` (already pulled in by `tseslint.configs.stylistic*`)
  disables conflicting ESLint rules. If formatting still flickers, check
  for a stale `.eslintrc` left over from a migration.
- **`endOfLine: 'lf'`** matters on Windows / WSL where `git config core.autocrlf`
  can rewrite line endings on checkout.

## Node runtime

- **`process.exit()` doesn't wait for pending I/O** (queued writes,
  unflushed logs). Use `process.exitCode = 1` and let the loop drain,
  or explicitly `await` flushes.
- **Unhandled promise rejections kill the process in Node 15+.** Old
  habits (rely on event listeners) break silently in modern Node. Use
  `process.on('unhandledRejection', ...)` for telemetry, not as a
  recovery mechanism.
- **`fetch` is global in Node 18+** — but the polyfill version in
  `node-fetch` has slightly different behavior around `keepalive`,
  `body` encoding. Stick to the global `fetch`; if you need quirks,
  use `undici` directly.
- **`AbortSignal.timeout(ms)`** (Node 17.3+) is cleaner than manually
  wiring `AbortController` + `setTimeout`. But it doesn't compose with
  `AbortSignal.any([...])` until Node 20.4+.

## Browser extensions

- **Manifest V3 service workers are not page workers.** They go to sleep
  after ~30s of inactivity. Don't store state in module-level vars
  expecting it to persist; use `chrome.storage` or `IndexedDB`.
- **`chrome.runtime.sendMessage` becomes `browser.runtime.sendMessage`
  on Firefox.** Use the `webextension-polyfill` or branch on the API
  presence.
- **The File System Access API is Chromium-only.** For Firefox, you
  need to fall back to download-as-file or `IndexedDB`.

## CI

- **`actions/setup-node@v4 cache: pnpm` requires `pnpm/action-setup`
  to run first.** Order matters in the workflow file.
- **`pnpm install --frozen-lockfile` in CI fails with a confusing
  message** when the lockfile is from a newer pnpm version than the one
  in CI. Pin pnpm version in CI to match local (or vice versa).
- **`actions/cache` keys based on lockfile hash** — if the lockfile
  isn't checked in, the cache thrashes. Always commit lockfiles.

## NPM publishing

- **`pnpm publish` doesn't run `prepublish` scripts** the way npm does.
  Use `prepublishOnly` (which both run) or call your build step
  explicitly in the release workflow.
- **`exports` map in `package.json` overrides `main` and `types`** for
  modern resolvers. Get this wrong and consumers see "Cannot find
  module 'your-package/something'". Test the published package locally
  with `pnpm pack` + install of the tarball.
- **`engines.node` is advisory by default.** Consumers don't error on
  mismatch unless they set `engine-strict=true`. Don't rely on it for
  correctness.
