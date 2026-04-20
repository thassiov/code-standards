# Go Project Standards

Self-contained standards for Go CLIs, daemons, and HTTP services.
Every template, config, and pattern needed to start a new project is
embedded in this document — no external file references required.

**Scope:** single-binary CLIs, daemons, and HTTP API servers in Go.
Not for shell scripts, one-shots, or experiments.

**How to use this doc:**
1. Read sections 1–3 for the mental model.
2. For a new project, work through the checklist in §16, copying the
   embedded templates verbatim and filling in the project name.
3. Sections 4–11 are the living rules. Re-read when unsure.
4. Section 15 is the list of things we've been burned by — consult
   before shipping.

---

## 1. Repo layout

```
<project>/
├── cmd/
│   └── <binary-name>/
│       └── main.go           # thin: just wires cobra, nothing else
├── internal/                  # everything that's not API surface
│   ├── config/
│   ├── logging/
│   ├── db/                    # or state/, store/
│   ├── <domain>/              # business logic, one dir per bounded area
│   └── ...
├── pkg/                       # ONLY if anything is meant to be imported externally
│   └── models/
├── api/                       # ONLY for HTTP services
│   ├── handlers/
│   ├── routes/
│   └── middleware/
├── tests/                     # integration tests (docker-compose lives here)
├── .github/workflows/
│   ├── ci.yml
│   └── release.yml
├── .golangci.yml
├── .gitignore
├── Makefile
├── go.mod
├── go.sum
├── README.md
└── LICENSE
```

Rules:
- **`cmd/<name>/main.go` is thin.** No business logic. It wires Cobra
  commands and exits. ~200 lines is the soft ceiling; beyond that,
  extra subcommands live in `cmd/<name>/cmd_<verb>.go` or equivalent,
  still in `package main`.
- **`internal/` is the default home for new code.** Only add to `pkg/`
  when another repo actually imports it.
- Don't nest `internal/` more than two levels deep. Three levels is a
  smell — split into sibling packages.
- One bounded concern per `internal/<dir>/` — e.g. `config`, `sync`,
  `embed`, `state`. Avoid dumping-ground names like `util`, `helpers`,
  `common`.

---

## 2. Go module & dependencies

- `go.mod` module path: match the actual import prefix. For GitHub-hosted
  personal code, `github.com/<user>/<project>`.
- **Go version: 1.25.x.** Pin in `.github/workflows/ci.yml` as
  `GO_VERSION: '1.25'`. Bump by editing one place.
- Core dependency set (don't re-evaluate per project):
  - `github.com/spf13/cobra` — CLI framework
  - `log/slog` (stdlib) — structured logging
  - `github.com/jackc/pgx/v5` — PostgreSQL client
  - `modernc.org/sqlite` — pure-Go SQLite (no CGO)
  - `github.com/go-chi/chi/v5` — HTTP router
- Run `go mod tidy` on every PR. CI checks.
- Vendor nothing unless you have a specific reason. `GOFLAGS=-mod=mod`.

---

## 3. Main entry pattern

Every `cmd/<name>/main.go` follows this skeleton. Copy verbatim, change
the `Use`, `Short`, and registered commands.

```go
package main

import (
    "context"
    "fmt"
    "os"
    "os/signal"
    "syscall"

    "github.com/spf13/cobra"
)

var (
    // Version is set at build time via ldflags.
    Version = "dev"
    // BuildTime is set at build time via ldflags.
    BuildTime = "unknown"
)

func main() {
    if err := run(); err != nil {
        fmt.Fprintf(os.Stderr, "error: %v\n", err)
        os.Exit(1)
    }
}

func run() error {
    rootCmd := &cobra.Command{
        Use:           "<name>",
        Short:         "<one-line description>",
        SilenceUsage:  true,  // don't print usage on every runtime error
        SilenceErrors: true,  // main() handles the error print
    }

    rootCmd.AddCommand(newVersionCmd())
    // ... register subcommands here ...

    ctx, cancel := signal.NotifyContext(context.Background(),
        syscall.SIGINT, syscall.SIGTERM)
    defer cancel()

    return rootCmd.ExecuteContext(ctx)
}

func newVersionCmd() *cobra.Command {
    return &cobra.Command{
        Use:   "version",
        Short: "Print version information",
        Run: func(_ *cobra.Command, _ []string) {
            fmt.Printf("version %s (built %s)\n", Version, BuildTime)
        },
    }
}
```

Why `main() → run() → cobra`:
- `run()` is testable (unit-test command wiring).
- Single error exit path, single error print format.
- `SilenceUsage` / `SilenceErrors` on the root prevent Cobra from
  dumping usage text on every runtime failure.

---

## 4. Cobra patterns

### 4.1 Subcommand factories

Each subcommand is built by a `newXxxCmd()` function returning
`*cobra.Command`. Keeps wiring declarative and lets `main.go` read
like a table of contents.

```go
func newDaemonCmd(configPath *string) *cobra.Command {
    return &cobra.Command{
        Use:   "daemon",
        Short: "Run the daemon",
        RunE: func(cmd *cobra.Command, _ []string) error {
            cfg, err := config.Load(*configPath)
            if err != nil {
                return err
            }
            // use cmd.Context() for cancellation
            return nil
        },
    }
}
```

Rules:
- Prefer `RunE` over `Run` — errors propagate to `main()`.
- Use `cmd.Context()` for ctx. Never a package-level global.
- Pass shared flags by pointer (`configPath *string`), not global
  vars, unless the flag is truly global (log level, verbose).
- Global vars in `main.go` are OK *only* for genuinely global things
  (`verbose`, loaded `cfg`, `logger`). Never in business-logic packages.

### 4.2 PersistentFlags + PersistentPreRunE

Put root-level flags on `PersistentFlags`; parse them once in
`PersistentPreRunE`. **Skip setup for commands that don't need it**
(version, help, completion) — otherwise `--version` dies when config
is missing.

```go
rootCmd.PersistentFlags().StringVarP(&configPath, "config", "c",
    config.DefaultConfigPath(), "path to config file")
rootCmd.PersistentFlags().BoolVarP(&verbose, "verbose", "v", false,
    "Enable verbose output")

rootCmd.PersistentPreRunE = func(cmd *cobra.Command, _ []string) error {
    switch cmd.Name() {
    case "version", "help", "completion":
        return nil
    }
    var err error
    cfg, err = config.Load(configPath)
    if err != nil {
        return fmt.Errorf("loading config: %w", err)
    }
    logger = logging.New(...)
    return nil
}
```

---

## 5. Config

One package (`internal/config`), one `Load()` function. Sources merge
in order: **defaults → file → env vars** (later overrides earlier).

```go
// internal/config/config.go
package config

import (
    "fmt"
    "os"
    "path/filepath"
)

type Config struct {
    MachineID string
    Scope     string
    Postgres  PostgresConfig
    // ...
}

type PostgresConfig struct {
    Host, User, Password, DB string
    Port                     int
}

func (p PostgresConfig) DSN() string {
    return fmt.Sprintf("postgres://%s:%s@%s:%d/%s",
        p.User, p.Password, p.Host, p.Port, p.DB)
}

func DefaultConfigPath() string {
    home, _ := os.UserHomeDir()
    return filepath.Join(home, ".config", "<binary-name>", "config.json")
}

func Load(path string) (*Config, error) {
    cfg := defaults()
    if err := loadFile(cfg, path); err != nil {
        return nil, fmt.Errorf("loading config file %s: %w", path, err)
    }
    loadEnv(cfg)
    if err := validate(cfg); err != nil {
        return nil, fmt.Errorf("invalid config: %w", err)
    }
    return cfg, nil
}
```

Rules:
- Each subsystem's config is its own struct (`PostgresConfig`,
  `OllamaConfig`) — not a flat bag of strings.
- Each subsystem exposes its own connection helper (`DSN()`,
  `BaseURL()`, `ListenAddr()`) rather than asking consumers to assemble
  the URL themselves.
- `Load()` validates and returns a complete config or an error. No
  partial configs escape into the wild.
- Secrets come from env vars or a secret manager. Never from a config
  file checked into git.

---

## 6. Logging

Use `log/slog`. No third-party logging libraries.

```go
logger := slog.New(slog.NewTextHandler(os.Stderr, &slog.HandlerOptions{
    Level: slog.LevelInfo,
}))

logger.Info("starting",
    "version", Version,
    "machine_id", cfg.MachineID,
)
```

Rules:
- One logger created in `PersistentPreRunE` or at the top of a daemon's
  run loop.
- Pass logger **by value** into structs that need it. Don't stash
  loggers in `context.Context` unless ctx is genuinely request-scoped
  (HTTP handlers).
- Handler choice: `TextHandler` for CLIs and local daemons,
  `JSONHandler` for services shipping logs to a collector.
- Log keys are lowercase snake_case: `machine_id`, `elapsed_ms`.
- Levels: `Debug` (chatty, off by default), `Info` (normal),
  `Warn` (recoverable), `Error` (action needed). No custom levels.
- Structured attrs over formatted strings. Do:
  `logger.Info("connected", "dsn", dsn)`, not
  `logger.Info(fmt.Sprintf("connected to %s", dsn))`.

---

## 7. Database

### 7.1 PostgreSQL (pgx v5)

Use `pgxpool`. Wrap pool creation to inject logger and timeouts:

```go
func Open(ctx context.Context, dsn string, logger *slog.Logger) (*pgxpool.Pool, error) {
    cfg, err := pgxpool.ParseConfig(dsn)
    if err != nil {
        return nil, fmt.Errorf("parsing DSN: %w", err)
    }
    cfg.MaxConns = 10
    cfg.MaxConnLifetime = 30 * time.Minute

    pool, err := pgxpool.NewWithConfig(ctx, cfg)
    if err != nil {
        return nil, fmt.Errorf("creating pool: %w", err)
    }
    if err := pool.Ping(ctx); err != nil {
        pool.Close()
        return nil, fmt.Errorf("pinging db: %w", err)
    }
    return pool, nil
}
```

### 7.2 SQLite (modernc.org/sqlite, pure Go)

Always set these pragmas. Without them, concurrent access (daemon +
CLI, daemon + another reader) corrupts state:

```go
func openSQLite(path string) (*sql.DB, error) {
    dsn := fmt.Sprintf("file:%s", path)
    db, err := sql.Open("sqlite", dsn)
    if err != nil {
        return nil, fmt.Errorf("opening sqlite %s: %w", path, err)
    }
    if _, err := db.Exec("PRAGMA busy_timeout = 5000"); err != nil {
        db.Close()
        return nil, fmt.Errorf("setting busy_timeout: %w", err)
    }
    if _, err := db.Exec("PRAGMA journal_mode = wal"); err != nil {
        db.Close()
        return nil, fmt.Errorf("setting journal_mode: %w", err)
    }
    if err := db.Ping(); err != nil {
        db.Close()
        return nil, fmt.Errorf("pinging sqlite %s: %w", path, err)
    }
    return db, nil
}
```

For read-only opens, append `?mode=ro` to the DSN.

---

## 8. HTTP servers

Use `go-chi/chi/v5`. Always set timeouts, always graceful-shutdown.

```go
func New(addr string, handler http.Handler, logger *slog.Logger) *http.Server {
    return &http.Server{
        Addr:              addr,
        Handler:           handler,
        ReadHeaderTimeout: 5 * time.Second,
        ReadTimeout:       30 * time.Second,
        WriteTimeout:      30 * time.Second,
        IdleTimeout:       120 * time.Second,
    }
}

// In run loop:
go func() {
    if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
        logger.Error("server stopped", "error", err)
    }
}()

<-ctx.Done()

shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
defer cancel()
return srv.Shutdown(shutdownCtx)
```

Handlers take dependencies as struct fields, not globals:

```go
type Handler struct {
    db     *pgxpool.Pool
    logger *slog.Logger
}

func (h *Handler) GetFoo(w http.ResponseWriter, r *http.Request) { ... }
```

Middleware ordering:
`chi.RequestID → chi.RealIP → Logger → Recoverer → Timeout → routes`.

---

## 9. Errors

- **Always wrap with `%w`.** Every error crossing a function boundary
  gets wrapped with context:
  ```go
  if err := foo(); err != nil {
      return fmt.Errorf("doing foo with %s: %w", name, err)
  }
  ```
- Use `errors.Is` / `errors.As` to inspect. Never string-match error
  messages.
- Don't log and return the same error. Pick one. Log at the top-level
  handler (cobra command, HTTP handler); return wrapped everywhere else.
- Define sentinel errors for cases callers need to branch on:
  ```go
  var ErrNotFound = errors.New("not found")
  ```
- No `panic` for recoverable conditions. `panic` only for programmer
  errors (impossible state, invariant violation).

---

## 10. Testing

- Unit tests live next to the code in `*_test.go`, same package. Use
  `package foo` for white-box tests, `package foo_test` for black-box.
- **Table-driven by default:**
  ```go
  func TestParse(t *testing.T) {
      tests := []struct {
          name    string
          input   string
          want    Result
          wantErr bool
      }{
          {"empty", "", Result{}, true},
          {"happy", "valid", Result{Value: 42}, false},
      }
      for _, tt := range tests {
          t.Run(tt.name, func(t *testing.T) {
              got, err := Parse(tt.input)
              if (err != nil) != tt.wantErr {
                  t.Fatalf("err = %v, wantErr = %v", err, tt.wantErr)
              }
              if !tt.wantErr && !reflect.DeepEqual(got, tt.want) {
                  t.Errorf("got %+v, want %+v", got, tt.want)
              }
          })
      }
  }
  ```
- Helpers named `newTestFoo(t *testing.T, ...)` — return the fixture
  and register cleanup via `t.Cleanup(...)`.
- Mock external APIs with `httptest.Server` — not by stubbing the
  client's methods. If the client uses HTTPS, use `httptest.NewTLSServer`.
- Integration tests under `tests/integration/` with the
  `//go:build integration` tag, invoked via `go test -tags=integration`.
  May depend on `tests/docker-compose.yml`.
- Always run with `-race` in CI. `go test -race ./...`.
- No `testify` unless you genuinely need it. The stdlib is enough for
  most assertions.

---

## 11. Linting — full `.golangci.yml`

Copy this verbatim into `.golangci.yml` at repo root.

```yaml
run:
  timeout: 5m
  tests: true

linters:
  enable:
    # Bugs & correctness
    - govet
    - staticcheck
    - gosimple
    - gosec
    - bodyclose
    - sqlclosecheck
    - rowserrcheck
    - nilerr
    - noctx
    - copyloopvar

    # Error handling
    - errcheck
    - errorlint
    - errname
    - wrapcheck

    # Complexity
    - gocyclo
    - gocognit
    - funlen
    - nestif

    # Duplication
    - dupl

    # Style & formatting
    - gofmt
    - goimports
    - revive
    - stylecheck
    - unconvert
    - unparam
    - misspell
    - godot
    - exhaustive

    # Performance
    - prealloc
    - gocritic

    # Dead code & unused
    - unused
    - ineffassign

    # Testing
    - tparallel

linters-settings:
  gocyclo:
    min-complexity: 15
  gocognit:
    min-complexity: 15
  funlen:
    lines: 80
    statements: 50
  nestif:
    min-complexity: 4
  dupl:
    threshold: 100
  exhaustive:
    default-signifies-exhaustive: true
  gosec:
    severity: medium
    confidence: medium
  errcheck:
    check-type-assertions: true
    check-blank: false
  errorlint:
    errorf: true
    asserts: true
    comparison: true
  wrapcheck:
    ignoreSigs:
      - .Errorf(
      - errors.New(
      - errors.Unwrap(
      - .Wrap(
      - .Wrapf(
      - .WithMessage(
      - .WithStack(
  revive:
    rules:
      - name: blank-imports
      - name: context-as-argument
      - name: context-keys-type
      - name: dot-imports
      - name: error-return
      - name: error-strings
      - name: error-naming
      - name: exported
      - name: if-return
      - name: increment-decrement
      - name: var-naming
      - name: var-declaration
      - name: package-comments
      - name: range
      - name: receiver-naming
      - name: time-naming
      - name: unexported-return
      - name: indent-error-flow
      - name: errorf
      - name: empty-block
      - name: superfluous-else
      - name: unused-parameter
      - name: unreachable-code
      - name: redefines-builtin-id
  gocritic:
    enabled-tags:
      - diagnostic
      - style
      - performance
    disabled-checks:
      - hugeParam
      - ifElseChain
  misspell:
    locale: US

issues:
  exclude-use-default: false
  max-issues-per-linter: 0
  max-same-issues: 0

  exclude-rules:
    # Test files — relax complexity, length, duplication, error checks
    - path: _test\.go
      linters:
        - funlen
        - gocyclo
        - gocognit
        - errcheck
        - dupl
        - unparam

    # Allow fmt.Print in main
    - path: cmd/
      linters:
        - forbidigo

    # Dot imports in tests (e.g. for testify)
    - path: _test\.go
      text: "dot-imports"

    # File paths from env/config are expected
    - linters:
        - gosec
      text: "G304"

    # wrapcheck and complexity are too strict for cobra commands
    - path: cmd/
      linters:
        - wrapcheck
        - gocognit
        - gocyclo

    # Long-running state machines have inherent complexity;
    # add paths for YOUR daemon/sync/state dirs here as they emerge.
    # Example:
    # - path: internal/(sync|extract)/
    #   linters:
    #     - gocognit
    #     - nestif

    # Command registration in main.go is inherently long
    - path: cmd/.*/main\.go
      linters:
        - funlen
```

Notes:
- Complexity budgets (`gocyclo: 15`, `funlen: 80/50`, `nestif: 4`) are
  deliberately tight. If you're hitting them, the code probably needs
  a rewrite, not an exemption.
- Add per-path exemptions only when a specific pattern (e.g. a
  state-machine loop) genuinely can't be simplified below the budget.

---

## 12. Build system — full `Makefile`

Copy and replace `<binary-name>` everywhere.

```make
.PHONY: all build test lint coverage security check clean install-local install-local-force dev fmt tidy
.PHONY: test-unit test-integration test-all ci docker-up docker-down help tools watch

# Build configuration
BINARY_DIR := bin
BINARY_NAME := <binary-name>
MAIN_PACKAGE := ./cmd/$(BINARY_NAME)
VERSION := $(shell git describe --tags --always --dirty 2>/dev/null || echo "dev")
BUILD_TIME := $(shell date -u '+%Y-%m-%dT%H:%M:%SZ')
LDFLAGS := -ldflags "-X main.Version=$(VERSION) -X main.BuildTime=$(BUILD_TIME)"

# Colors
GREEN := \033[0;32m
YELLOW := \033[0;33m
RED := \033[0;31m
NC := \033[0m

all: check build

# =============================================================================
# Build
# =============================================================================

build:
	@mkdir -p $(BINARY_DIR)
	go build $(LDFLAGS) -o $(BINARY_DIR)/$(BINARY_NAME) $(MAIN_PACKAGE)
	@echo "$(GREEN)Built: $(BINARY_DIR)/$(BINARY_NAME)$(NC)"

# =============================================================================
# Test
# =============================================================================

test-unit:
	@echo "$(GREEN)Running unit tests...$(NC)"
	go test -v -race ./...

# Requires Docker; skip unless you have integration tests
test-integration: docker-up
	@echo "$(GREEN)Running integration tests...$(NC)"
	cd tests && docker compose run --rm test-runner go test -v -tags=integration ./tests/integration/...
	$(MAKE) docker-down

test-all: test-unit test-integration
	@echo "$(GREEN)All tests passed!$(NC)"

test: test-unit

# =============================================================================
# Code quality
# =============================================================================

fmt:
	@echo "$(GREEN)Formatting code...$(NC)"
	go fmt ./...
	@which goimports > /dev/null 2>&1 && goimports -w . || echo "$(YELLOW)goimports not installed, skipping$(NC)"

tidy:
	go mod tidy

vet:
	@echo "$(GREEN)Running go vet...$(NC)"
	go vet ./...

lint:
	@echo "$(GREEN)Running golangci-lint...$(NC)"
	@which golangci-lint > /dev/null 2>&1 && golangci-lint run ./... || echo "$(YELLOW)golangci-lint not installed, running go vet instead$(NC)" && go vet ./...

security:
	@echo "$(GREEN)Running security checks...$(NC)"
	@which gosec > /dev/null 2>&1 && gosec -quiet ./... || echo "$(YELLOW)gosec not installed, skipping$(NC)"

coverage:
	@echo "$(GREEN)Running tests with coverage...$(NC)"
	go test -coverprofile=coverage.out -covermode=atomic ./...
	@echo ""
	@echo "$(GREEN)Coverage summary:$(NC)"
	@go tool cover -func=coverage.out | grep -E "^total:|internal/"
	@echo ""
	@go tool cover -html=coverage.out -o coverage.html
	@echo "$(GREEN)Full report: coverage.html$(NC)"

# =============================================================================
# CI pipelines
# =============================================================================

ci: fmt tidy vet lint test-unit coverage build
	@echo ""
	@echo "$(GREEN)========================================$(NC)"
	@echo "$(GREEN)  CI Pipeline completed successfully!  $(NC)"
	@echo "$(GREEN)========================================$(NC)"

check: fmt tidy vet test-unit build
	@echo "$(GREEN)Quick check passed!$(NC)"

# =============================================================================
# Docker (only if you have tests/docker-compose.yml)
# =============================================================================

docker-up:
	cd tests && docker compose up -d
	@sleep 5

docker-down:
	cd tests && docker compose down

# =============================================================================
# Utility
# =============================================================================

clean:
	rm -rf $(BINARY_DIR)
	rm -f coverage.out coverage.html

install-local: build
	@mkdir -p ~/.local/bin
	@if [ -x ~/.local/bin/$(BINARY_NAME) ]; then \
		echo "$(YELLOW)Current:$(NC) $$(~/.local/bin/$(BINARY_NAME) version 2>/dev/null || echo 'unknown')"; \
		echo "$(GREEN)New:$(NC)     $(BINARY_NAME) version $(VERSION) (built $(BUILD_TIME))"; \
		echo ""; \
		read -p "Replace existing installation? [y/N] " confirm; \
		if [ "$$confirm" != "y" ] && [ "$$confirm" != "Y" ]; then \
			echo "$(YELLOW)Cancelled$(NC)"; \
			exit 1; \
		fi; \
	else \
		echo "$(GREEN)Installing:$(NC) $(BINARY_NAME) version $(VERSION) (built $(BUILD_TIME))"; \
	fi
	cp $(BINARY_DIR)/$(BINARY_NAME) ~/.local/bin/
	@echo "$(GREEN)Installed to ~/.local/bin/$(BINARY_NAME)$(NC)"

install-local-force: build
	@mkdir -p ~/.local/bin
	cp $(BINARY_DIR)/$(BINARY_NAME) ~/.local/bin/
	@echo "$(GREEN)Installed to ~/.local/bin/$(BINARY_NAME)$(NC)"

dev: build
	./$(BINARY_DIR)/$(BINARY_NAME)

watch:
	find . -name '*.go' | entr -c make build

tools:
	@echo "$(GREEN)Installing development tools...$(NC)"
	go install golang.org/x/tools/cmd/goimports@latest
	go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest
	go install github.com/securego/gosec/v2/cmd/gosec@latest

help:
	@echo "Available targets:"
	@echo "  build          Build the binary"
	@echo "  install-local  Build and install to ~/.local/bin"
	@echo "  test           Run unit tests (alias for test-unit)"
	@echo "  test-unit      Run unit tests with race detector"
	@echo "  test-integration  Run integration tests (requires Docker)"
	@echo "  coverage       Generate coverage report"
	@echo "  fmt            Format code"
	@echo "  tidy           go mod tidy"
	@echo "  vet            go vet"
	@echo "  lint           golangci-lint"
	@echo "  security       gosec"
	@echo "  check          fmt + tidy + vet + test + build (fast local)"
	@echo "  ci             Full CI pipeline"
	@echo "  clean          Remove build artifacts"
	@echo "  tools          Install dev tools"
	@echo "  help           Show this help"
```

**Ldflags for version injection are mandatory.** `<binary> version`
should always print the real commit + build timestamp.

---

## 13. CI — full `.github/workflows/ci.yml`

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  GO_VERSION: '1.25'

jobs:
  lint:
    name: Lint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: ${{ env.GO_VERSION }}
          cache: true
      - name: golangci-lint
        uses: golangci/golangci-lint-action@v4
        with:
          version: latest
          args: --timeout=5m

  test:
    name: Test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: ${{ env.GO_VERSION }}
          cache: true
      - name: Run tests
        run: go test -v ./...
      - name: Run tests with race detector
        run: go test -race ./...

  coverage:
    name: Coverage
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: ${{ env.GO_VERSION }}
          cache: true
      - name: Generate coverage
        run: go test -coverprofile=coverage.out -covermode=atomic ./...
      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v4
        with:
          files: ./coverage.out
          fail_ci_if_error: false

  build:
    name: Build
    runs-on: ubuntu-latest
    strategy:
      matrix:
        goarch: [amd64, arm64]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: ${{ env.GO_VERSION }}
          cache: true
      - name: Build
        env:
          GOOS: linux
          GOARCH: ${{ matrix.goarch }}
        run: |
          mkdir -p dist
          go build -ldflags "-X main.Version=${{ github.sha }} -X main.BuildTime=$(date -u '+%Y-%m-%dT%H:%M:%SZ')" \
            -o dist/<binary-name>-linux-${{ matrix.goarch }} ./cmd/<binary-name>

  security:
    name: Security
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: ${{ env.GO_VERSION }}
          cache: true
      - name: Install gosec
        run: go install github.com/securego/gosec/v2/cmd/gosec@latest
      - name: Run gosec
        run: gosec -quiet ./...
```

**`.github/workflows/release.yml`** (only needed when cutting versioned
releases; tag as `vX.Y.Z` to trigger):

```yaml
name: Release

on:
  push:
    tags:
      - 'v*'

env:
  GO_VERSION: '1.25'

permissions:
  contents: write

jobs:
  test:
    name: Test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: ${{ env.GO_VERSION }}
          cache: true
      - name: Run tests
        run: go test -v ./...
      - name: Run tests with race detector
        run: go test -race ./...

  release:
    name: Release
    needs: test
    runs-on: ubuntu-latest
    strategy:
      matrix:
        goarch: [amd64, arm64]
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: actions/setup-go@v5
        with:
          go-version: ${{ env.GO_VERSION }}
          cache: true
      - name: Build
        env:
          GOOS: linux
          GOARCH: ${{ matrix.goarch }}
        run: |
          mkdir -p dist
          go build -ldflags "-X main.Version=${{ github.ref_name }} -X main.BuildTime=$(date -u '+%Y-%m-%dT%H:%M:%SZ')" \
            -o dist/<binary-name>-linux-${{ matrix.goarch }} ./cmd/<binary-name>
      - name: Upload binaries to release
        uses: softprops/action-gh-release@v2
        with:
          files: dist/*
          generate_release_notes: true
```

---

## 14. Service deployment (daemons only)

For daemons that should run as a systemd user unit, include the
install step in the Makefile. Template for the service file:

```ini
[Unit]
Description=<one-line description>
After=<dependency.service>

[Service]
Type=notify
ExecStart=%h/.local/bin/<binary-name> daemon
Restart=on-failure
RestartSec=10
WatchdogSec=120
Environment=HOME=%h

[Install]
WantedBy=default.target
```

Install via `systemctl --user enable --now <binary-name>.service`.
**User units only** — don't install system-level services.

---

## 15. Gotchas — stuff we've been burned by

- **Cobra `PersistentPreRunE` runs for *everything*, including `--help`
  and `--version`.** If it unconditionally loads config, `--version`
  dies on a machine with no config. Always early-return on
  `version` / `help` / `completion`.
- **SQLite without `busy_timeout` + `journal_mode=WAL`** → `database is
  locked` in a daemon + CLI setup. Every time. Not optional.
- **`wrapcheck` false positives** when wrapping errors from your own
  helpers. Either add to `ignoreSigs` or return pre-wrapped.
- **`httptest.NewServer` vs `NewTLSServer`** — if the client is
  hard-coded to HTTPS, the plain version gives an unhelpful
  "unknown protocol" error. Use the TLS variant.
- **`go-chi`'s `Timeout` middleware doesn't cancel the handler's
  work** — it only cancels the response write. If you have expensive
  work, check `r.Context().Done()` yourself.
- **`signal.NotifyContext` only covers the current process.** If you
  `os/exec` a subprocess, propagate the context via
  `exec.CommandContext` or the child won't die on Ctrl-C.
- **`modernc.org/sqlite` is pure Go but slower than CGO alternatives.**
  For hot paths (embedding stores, high-QPS), profile before assuming
  it's fine.
- **`go build` without ldflags** produces a binary that reports
  `version dev (built unknown)`. Always build through the Makefile in
  local dev, or set `LDFLAGS` explicitly.
- **`go test` without `-race`** passes locally but CI catches races.
  Run `-race` locally too.
- **Test parallelism + shared state.** `t.Parallel()` on tests that
  mutate shared globals or touch the same DB will flake
  nondeterministically. Either isolate state per test or don't
  parallelize that test.

---

## 16. Checklist for a new project

Work through in order. Each step has a template earlier in this doc.

- [ ] Create repo, `go mod init github.com/<user>/<project>`
- [ ] Pin Go version in CI (`GO_VERSION: '1.25'`)
- [ ] `cmd/<name>/main.go` from §3 skeleton
- [ ] `internal/config/config.go` with `Load()` + `DefaultConfigPath()` (§5)
- [ ] `internal/logging/logging.go` if the one-liner slog isn't enough
- [ ] `.golangci.yml` from §11 verbatim
- [ ] `Makefile` from §12, replace `<binary-name>`
- [ ] `.github/workflows/ci.yml` from §13, replace `<binary-name>`
- [ ] `.github/workflows/release.yml` if you plan to cut tagged releases
- [ ] `.gitignore` covers:
  ```gitignore
  # Binaries
  /bin/
  /dist/

  # Test / coverage output
  coverage.out
  coverage.html
  *.test

  # Editor
  .vscode/
  .idea/
  *.swp

  # OS
  .DS_Store
  ```
- [ ] `README.md`: what it does, install, usage, config, dev
- [ ] `LICENSE`
- [ ] `make tools` to install dev tools
- [ ] First commit should run `make ci` clean before push

---

## 17. What's intentionally NOT here

- **Observability / metrics.** No canonical Prometheus pattern yet.
  Add when one exists.
- **Secrets management.** Varies by environment (1Password `op` CLI,
  Vault, AWS SM, plain env). Pick per-project; don't put secrets in
  config files regardless.
- **Frontend / UI.** Out of scope.
- **SQL migrations.** No preferred tool yet (golang-migrate vs sqlc vs
  atlas). Pick per-project, document in the project's README.
- **Dockerfile.** Write one when you need it; no canonical template
  worth enforcing.
