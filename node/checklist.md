# New Project Checklist

Step-by-step. Each step has a template earlier in these docs.

## 1. Project skeleton

- [ ] `mkdir <project> && cd <project>`
- [ ] `git init -b main`
- [ ] Decide: app or library? Mono or single-package? CLI / API / browser ext?

## 2. Files to create up front

- [ ] `.nvmrc` — `22`
- [ ] `.gitignore` — copy from [`tooling.md` §5](tooling.md#5-gitignore)
- [ ] `package.json` — start minimal, fill in as deps land:
      ```json
      {
        "name": "<project>",
        "version": "0.0.0",
        "private": true,
        "type": "module",
        "engines": { "node": ">=22.0.0" },
        "scripts": {}
      }
      ```
      For library publish, set `"private": false`, add `main`/`types`/`exports`.
- [ ] `tsconfig.json` — copy from [`tooling.md` §1](tooling.md#1-tsconfigjson--base)
- [ ] `eslint.config.js` — copy from [`tooling.md` §2](tooling.md#2-eslint--flat-config)
- [ ] `.prettierrc.json` + `.prettierignore` — copy from [`tooling.md` §3](tooling.md#3-prettier)
- [ ] `vitest.config.ts` — copy from [`tests.md` §3](tests.md#3-vitest-config)
- [ ] `README.md` — copy template from [`docs.md` §1](docs.md#1-readme--minimum-viable)
- [ ] `LICENSE` — pick one; MIT is the default

## 3. Install dependencies

- [ ] Pick a package manager — pnpm by default
- [ ] Dev dependencies (foundation):
      ```bash
      pnpm add -D typescript tsx vitest @vitest/coverage-v8 \
        eslint typescript-eslint @eslint/js \
        eslint-plugin-import-x eslint-plugin-promise eslint-plugin-unicorn \
        prettier eslint-config-prettier \
        @types/node
      ```
- [ ] Project-type dependencies — see the relevant `topics/*.md`

## 4. Scripts

- [ ] Copy script block from [`tooling.md` §4](tooling.md#4-package-scripts)

## 5. CI

- [ ] `mkdir -p .github/workflows`
- [ ] `.github/workflows/ci.yml` — copy from [`ci.md` §1](ci.md#1-githubworkflowsciyml)
- [ ] If the project is published to npm: add `release.yml` from [`ci.md` §2](ci.md#2-githubworkflowsreleaseyml)

## 6. First commit

- [ ] Verify everything works locally:
      ```bash
      pnpm install
      pnpm ci         # runs format:check + lint + typecheck + test + build
      ```
- [ ] At least one source file in `src/` (a placeholder `index.ts` is fine)
- [ ] At least one test in `*.test.ts` — even a trivial one. Empty test
      suites give false reassurance.
- [ ] `git add . && git commit -m "initial commit"`
- [ ] Push to GitHub.

## 7. Branch protection

- [ ] In repo Settings → Branches: require `format`, `lint`, `typecheck`,
      `test`, `build` to pass before merge to `main`.
- [ ] Disallow force pushes and deletions on `main`.

## 8. Project-type extras

For the project type, add the steps from the relevant topic file:

- [ ] CLI → [`topics/cli.md`](topics/cli.md)
- [ ] REST API → [`topics/api-rest.md`](topics/api-rest.md)
- [ ] Browser extension → [`topics/browser-extension.md`](topics/browser-extension.md)
- [ ] Containerized → [`topics/containers.md`](topics/containers.md)
- [ ] AWS deployment → [`topics/aws.md`](topics/aws.md)

## 9. First proper feature

Now that the scaffolding is in place, the first real piece of work
should:

- [ ] Add a real module under `src/<bounded-concern>/`
- [ ] Write at least one unit test for it
- [ ] Update the README's "Usage" section
- [ ] If it required a non-trivial decision, write an ADR — see [`docs.md` §3](docs.md#3-adrs-architecture-decision-records)

## 10. Sanity check

Before considering the scaffold "done":

- [ ] `pnpm ci` passes locally
- [ ] CI passes on the first push
- [ ] `pnpm build` produces something runnable (even if it's just a stub)
- [ ] README's quick-start works for someone who just cloned the repo
