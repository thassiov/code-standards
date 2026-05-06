# Tooling

TypeScript, ESLint, Prettier, package scripts. Templates ready to copy.

## 1. `tsconfig.json` — base

Copy verbatim, adjust `outDir`/`rootDir` per project layout.

```jsonc
{
  "compilerOptions": {
    "target": "ES2023",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "lib": ["ES2023"],
    "outDir": "dist",
    "rootDir": "src",

    // Strictness — non-negotiable
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitOverride": true,
    "noFallthroughCasesInSwitch": true,
    "noPropertyAccessFromIndexSignature": true,

    // Hygiene
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "skipLibCheck": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "verbatimModuleSyntax": true,

    // Output
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "**/*.test.ts"]
}
```

For browser/extension targets, swap:
- `"target": "ES2022"` (broader compat)
- `"module": "ESNext"` + `"moduleResolution": "Bundler"`
- `"lib": ["ES2022", "DOM", "DOM.Iterable"]`
- Add `"jsx": "react-jsx"` if React/JSX is used.

For test compilation, separate `tsconfig.test.json` extending the base
with `"include": ["src/**/*"]` (no exclude) is the simplest split.

## 2. ESLint — flat config

Use ESLint v9 flat config. Stack: `@typescript-eslint`, `eslint-plugin-import-x`,
`eslint-plugin-promise`, `eslint-plugin-unicorn`.

`eslint.config.js`:

```js
import js from '@eslint/js';
import tseslint from 'typescript-eslint';
import importPlugin from 'eslint-plugin-import-x';
import promisePlugin from 'eslint-plugin-promise';
import unicorn from 'eslint-plugin-unicorn';

export default tseslint.config(
  // 1. Apply to TS files
  {
    files: ['**/*.{ts,tsx}'],
    languageOptions: {
      ecmaVersion: 2023,
      sourceType: 'module',
      parserOptions: {
        project: ['./tsconfig.json'],
      },
    },
  },

  // 2. Recommended bases
  js.configs.recommended,
  ...tseslint.configs.strictTypeChecked,
  ...tseslint.configs.stylisticTypeChecked,
  importPlugin.flatConfigs.recommended,
  importPlugin.flatConfigs.typescript,
  promisePlugin.configs['flat/recommended'],
  unicorn.configs.recommended,

  // 3. Project rules
  {
    rules: {
      // Style
      '@typescript-eslint/consistent-type-imports': ['error', { prefer: 'type-imports' }],
      '@typescript-eslint/no-import-type-side-effects': 'error',
      '@typescript-eslint/no-explicit-any': 'error',
      '@typescript-eslint/no-non-null-assertion': 'error',
      '@typescript-eslint/no-unused-vars': ['error', { argsIgnorePattern: '^_', varsIgnorePattern: '^_' }],

      // Promises
      '@typescript-eslint/no-floating-promises': 'error',
      '@typescript-eslint/no-misused-promises': 'error',
      'promise/always-return': 'error',
      'promise/catch-or-return': 'error',

      // Imports
      'import-x/order': ['error', {
        groups: ['builtin', 'external', 'internal', 'parent', 'sibling', 'index'],
        'newlines-between': 'always',
        alphabetize: { order: 'asc', caseInsensitive: true },
      }],
      'import-x/no-default-export': 'warn', // CommonJS-only files exempted below
      'import-x/no-cycle': 'error',

      // Unicorn — opinionated style
      'unicorn/filename-case': ['error', { case: 'kebabCase' }],
      'unicorn/prefer-node-protocol': 'error',
      'unicorn/no-null': 'off', // null is fine
      'unicorn/prevent-abbreviations': 'off', // too noisy
    },
  },

  // 4. Test files — relax some rules
  {
    files: ['**/*.test.ts', '**/*.spec.ts', 'test/**/*.ts'],
    rules: {
      '@typescript-eslint/no-non-null-assertion': 'off',
      '@typescript-eslint/no-unsafe-assignment': 'off',
      '@typescript-eslint/no-unsafe-call': 'off',
      '@typescript-eslint/no-unsafe-member-access': 'off',
    },
  },

  // 5. Ignore
  {
    ignores: ['dist/**', 'coverage/**', 'node_modules/**', '.next/**'],
  },
);
```

Notes:
- **`strictTypeChecked` requires the `parserOptions.project` setting.**
  Slow on big repos; consider sub-tsconfigs for tests vs source.
- The `import-x/no-default-export: warn` is a tradeoff — default exports
  break refactor-rename and dead-code analysis; named exports are
  uniformly better. The exception is files where a framework requires
  default exports (Next.js pages, some Vite plugins).

## 3. Prettier

`.prettierrc.json`:

```json
{
  "semi": true,
  "singleQuote": true,
  "trailingComma": "all",
  "printWidth": 100,
  "tabWidth": 2,
  "arrowParens": "always",
  "endOfLine": "lf"
}
```

Don't enable Prettier rules in ESLint — let Prettier own formatting,
ESLint own correctness. `eslint-config-prettier` (already pulled in via
`tseslint.configs.stylistic*`) disables conflicting ESLint rules.

`.prettierignore`:

```
dist
coverage
.next
pnpm-lock.yaml
*.min.js
```

## 4. Package scripts

Standard `package.json` scripts. Keep the names consistent across all
projects so muscle memory works:

```json
{
  "scripts": {
    "build": "tsc",
    "dev": "tsx watch src/index.ts",
    "start": "node dist/index.js",

    "test": "vitest run",
    "test:watch": "vitest",
    "test:coverage": "vitest run --coverage",

    "lint": "eslint .",
    "lint:fix": "eslint . --fix",

    "format": "prettier --write .",
    "format:check": "prettier --check .",

    "typecheck": "tsc --noEmit",

    "ci": "pnpm format:check && pnpm lint && pnpm typecheck && pnpm test && pnpm build"
  }
}
```

The `ci` script is the local equivalent of the CI pipeline — fast
feedback before pushing. CI runs the same steps individually for better
log output (see [`ci.md`](ci.md)).

## 5. `.gitignore`

```gitignore
# Dependencies
node_modules/
.pnpm-store/

# Build output
dist/
build/
.next/
out/

# Coverage
coverage/
*.lcov

# Environment
.env
.env.local
.env.*.local

# Logs
*.log
npm-debug.log*
pnpm-debug.log*
yarn-debug.log*
yarn-error.log*

# Editor
.vscode/
.idea/
*.swp

# OS
.DS_Store
Thumbs.db

# Misc
*.tsbuildinfo
.eslintcache
```

## 6. `.nvmrc`

```
22
```

That's it — one line, the major version. Pin the minor only if you have
a specific dependency on Node features only in 22.x.

## 7. Pre-commit / pre-push (optional)

Lefthook is the lightest option. `lefthook.yml`:

```yaml
pre-commit:
  parallel: true
  commands:
    lint:
      glob: '*.{ts,tsx,js}'
      run: pnpm exec eslint --fix {staged_files} && git add {staged_files}
    format:
      glob: '*.{ts,tsx,js,md,json,yaml,yml}'
      run: pnpm exec prettier --write {staged_files} && git add {staged_files}

pre-push:
  commands:
    typecheck:
      run: pnpm typecheck
    test:
      run: pnpm test
```

Skip this entirely on small projects; `pnpm ci` before push is sufficient.

## 8. Static analysis stack

`tsc` and ESLint are not enough. The Go side runs golangci-lint with a
~15-tool stack (govet, staticcheck, gosec, errcheck, unused, gocritic, …).
Mirror that spirit on the Node side: every project layers the tools below
on top of `tsc` + ESLint, each one wired into `pnpm static` so a single
script runs the full suite.

| Tool | Role | What it catches | Go analog |
|------|------|-----------------|-----------|
| **prettier** (`format:check`) | formatter | inconsistent style | `gofmt` |
| **eslint** (`lint`) | linter | bugs, style, type-aware patterns | `go vet` + most of staticcheck |
| **eslint-plugin-sonarjs** | inside eslint | cognitive complexity, code smells | gocritic + cyclop |
| **tsc --noEmit** (`typecheck`) | type checker | type errors | the compiler itself |
| **knip** (`knip`) | unused-code finder | dead exports, files, devDependencies | `staticcheck unused` + `deadcode` |
| **madge** (`deps:circular`) | dep graph | circular imports | (no direct equivalent) |
| **dependency-cruiser** (`deps:boundaries`) | architectural rules | layering violations | `internal/` package boundaries |
| **type-coverage** (`type-coverage`) | typing strictness | drops in % typed below threshold | (the no-`any` discipline) |
| **pnpm audit** (`audit`) | supply-chain | known CVEs in deps | `gosec` (different angle) |

Wire all of these into one `pnpm static`, and chain `static` + `test` +
`build` into `pnpm verify` (NOT `pnpm ci` — pnpm reserves that name and
will return `ERR_PNPM_CI_NOT_IMPLEMENTED`). CI runs `pnpm verify`.

### Required `package.json` scripts

```jsonc
{
  "scripts": {
    "format:check":   "prettier --check .",
    "lint":           "eslint .",
    "typecheck":      "tsc --noEmit",
    "knip":           "knip",
    "deps:circular":  "madge --circular --extensions ts,tsx src",
    "deps:boundaries":"depcruise --config .dependency-cruiser.cjs src",
    "type-coverage":  "type-coverage --strict --at-least 99",
    "audit":          "pnpm audit --prod --audit-level high",
    "test":           "vitest run",
    "build":          "...",
    "static":         "pnpm format:check && pnpm lint && pnpm typecheck && pnpm knip && pnpm deps:circular && pnpm deps:boundaries && pnpm type-coverage",
    "verify":         "pnpm static && pnpm test && pnpm build"
  }
}
```

### Complexity gates (eslint built-ins + sonarjs)

Add these to the eslint config's `rules` block. They mirror the Go
defaults — strict but generous:

```js
complexity:                       ['error', { max: 15 }],
'max-depth':                      ['error', { max: 4 }],
'max-lines-per-function':         ['error', { max: 120, skipBlankLines: true, skipComments: true, IIFEs: true }],
'max-lines':                      ['error', { max: 500, skipBlankLines: true, skipComments: true }],
'max-statements':                 ['error', { max: 30 }],
'max-nested-callbacks':           ['error', { max: 4 }],
'max-params':                     ['error', { max: 5 }],
'sonarjs/cognitive-complexity':   ['error', 15],
```

In test files, disable `max-lines`, `max-lines-per-function`,
`max-statements`, and `sonarjs/cognitive-complexity` — `describe`
blocks are inherently large.

### dependency-cruiser as the `internal/` analog

Folder structure suggests architecture; dependency-cruiser enforces it.
Treat each top-level folder under `src/` as a Go package: declare which
folders may import which. Example for a browser extension with a typed
message bus:

- `shared/` — leaf-ish (types, message bus). May not import UI/runtime layers.
- `storage/` — pure persistence. May import only `shared/` + `lib/`.
- `content/` — page-injected; cannot import background/options/popup/snippets/storage.
- `options/` and `popup/` — UI; talk to SW only via `shared/send.ts`.
- `background/` — composition root.
- `lib/` — utilities only; no app-shaped imports.

Set `tsPreCompilationDeps: true` so type-only imports are still
recognized as dependencies — otherwise files that only export types
look like orphans.

### Per-file complexity exemptions

Known offenders go in named overrides at the bottom of `eslint.config.js`,
each with a one-line justification (or a tracking ticket):

```js
{
  files: ['src/content/read-selection.ts'],
  rules: {
    complexity: 'off',
    'max-statements': 'off',
    'sonarjs/cognitive-complexity': 'off',
  },
},
```

Do NOT silence complexity rules globally. New code must clear the
gates; debt is opted into explicitly, file by file, and tracked.

### Sonarjs noise to disable up front

Several sonarjs rules don't pay rent. Turn off:

- `sonarjs/void-use` — `void promise` is the canonical fire-and-forget marker.
- `sonarjs/prefer-read-only-props` — local Preact/React component prop types don't need `Readonly<>`.
- `sonarjs/no-alphabetical-sort` — `Array#toSorted()` is fine without a `localeCompare` for pure string lists.
- `sonarjs/argument-type` — false-flags `Array#slice(start, undefined)`.

Keep the rest of `sonarjs.configs.recommended`.

### What to disable in unicorn

`eslint-plugin-unicorn` is opinionated. The defaults that fight the
codebase more than they help:

- `unicorn/catch-error-name` — pins to `error_`; conflicts with the `err` convention.
- `unicorn/no-useless-undefined` — conflicts with TS argument inference (`mockResolvedValue(undefined)`).
- `unicorn/consistent-function-scoping` and `unicorn/no-await-expression-member` — disable for tests only.

### import-x resolver

Install `eslint-import-resolver-typescript` and configure it explicitly,
otherwise every external import flags as unresolved:

```js
settings: {
  'import-x/resolver': {
    typescript: { project: './tsconfig.json' },
    node: true,
  },
},
```

### What to leave OUT of eslint's purview

Files not in `tsconfig.include` (vite/vitest/playwright configs, helper
scripts) cause type-aware lint to throw. Add them to the eslint
`ignores` list — they don't ship at runtime and rarely need linting.
For JS config files that *are* linted, add an override applying
`tseslint.configs.disableTypeChecked` so type-aware rules don't try to
load them in the TS project.
