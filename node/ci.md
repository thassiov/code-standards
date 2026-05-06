# CI

GitHub Actions workflow templates. Each job maps 1:1 to a `package.json`
script so the script runner is the source of truth and CI/local can't
drift.

## 1. `.github/workflows/ci.yml`

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  NODE_VERSION: '22'
  PNPM_VERSION: '9'

jobs:
  install:
    name: Install
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
        with:
          version: ${{ env.PNPM_VERSION }}
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: pnpm
      - run: pnpm install --frozen-lockfile

  format:
    name: Format check
    runs-on: ubuntu-latest
    needs: install
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
        with:
          version: ${{ env.PNPM_VERSION }}
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: pnpm
      - run: pnpm install --frozen-lockfile
      - run: pnpm format:check

  lint:
    name: Lint
    runs-on: ubuntu-latest
    needs: install
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
        with:
          version: ${{ env.PNPM_VERSION }}
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: pnpm
      - run: pnpm install --frozen-lockfile
      - run: pnpm lint

  typecheck:
    name: Type check
    runs-on: ubuntu-latest
    needs: install
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
        with:
          version: ${{ env.PNPM_VERSION }}
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: pnpm
      - run: pnpm install --frozen-lockfile
      - run: pnpm typecheck

  test:
    name: Test
    runs-on: ubuntu-latest
    needs: install
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
        with:
          version: ${{ env.PNPM_VERSION }}
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: pnpm
      - run: pnpm install --frozen-lockfile
      - run: pnpm test:coverage
      - uses: actions/upload-artifact@v4
        with:
          name: coverage
          path: coverage/

  build:
    name: Build
    runs-on: ubuntu-latest
    needs: [format, lint, typecheck, test]
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
        with:
          version: ${{ env.PNPM_VERSION }}
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: pnpm
      - run: pnpm install --frozen-lockfile
      - run: pnpm build
      - uses: actions/upload-artifact@v4
        with:
          name: dist
          path: dist/
```

The DAG: `install` → 4 parallel checks → `build`. Build only runs if
the four checks pass. Each job re-installs because GH Actions doesn't
share `node_modules` across jobs by default — relying on the pnpm
cache keeps it fast (~5 sec on a cached run).

## 2. `.github/workflows/release.yml`

For npm-published packages (libraries). Skip for apps.

```yaml
name: Release

on:
  push:
    tags:
      - 'v*'

permissions:
  contents: write
  id-token: write   # required for npm provenance

jobs:
  test:
    uses: ./.github/workflows/ci.yml

  publish:
    name: Publish to npm
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
        with:
          version: '9'
      - uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: pnpm
          registry-url: 'https://registry.npmjs.org'
      - run: pnpm install --frozen-lockfile
      - run: pnpm build
      - run: pnpm publish --access public --provenance --no-git-checks
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}

      - name: Create GitHub release
        uses: softprops/action-gh-release@v2
        with:
          generate_release_notes: true
```

## 3. Caching

- `actions/setup-node@v4` with `cache: pnpm` handles the pnpm-store cache
  automatically — keyed off `pnpm-lock.yaml`.
- For TypeScript build cache (`*.tsbuildinfo`), add an explicit cache step:

  ```yaml
  - uses: actions/cache@v4
    with:
      path: '**/*.tsbuildinfo'
      key: ${{ runner.os }}-tsbuildinfo-${{ hashFiles('**/tsconfig*.json', 'src/**/*.ts') }}
  ```

  Only worth it on big projects (>30s typecheck). Small projects don't
  recoup the cache restore time.

## 4. Branch protection

GH branch protection on `main`:
- Require status checks: `format`, `lint`, `typecheck`, `test`, `build`
- Require branches up to date before merging
- Require conversation resolution
- Disallow force pushes
- Disallow deletions

Set this once in the repo settings. `.github/CODEOWNERS` is optional —
useful only on multi-author repos.

## 5. Secrets

- Never commit secrets. Lockfile + `.gitignore` keep `.env` out.
- CI secrets via repo or org-level secrets, accessed as
  `${{ secrets.NAME }}`.
- For npm: `NPM_TOKEN` (automation token, not classic).
- For Codecov: `CODECOV_TOKEN` (only if private repo).

## 6. What's intentionally NOT in CI

- Running E2E (Playwright) — needs a real stack; runs on staging deploys
  via a separate workflow when the time comes.
- Running integration tests against external services — keep CI hermetic.
- Auto-merging Renovate/Dependabot PRs — manual review for now.
