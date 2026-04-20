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
│   ├── db/                    # or state/, store/
│   ├── <domain>/              # business logic, one dir per bounded area
│   └── ...
├── pkg/                       # ONLY if anything is meant to be imported externally
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
- **Go version: 1.25.x.** The `go` directive in `go.mod` and
  `GO_VERSION` in CI must match. If `go mod init` writes a newer version
  (because your local toolchain is newer), edit `go.mod` back to `go 1.25`.
- Core dependency set (don't re-evaluate per project):
  - `github.com/spf13/cobra` — CLI framework
  - `log/slog` (stdlib) — structured logging
  - `github.com/jackc/pgx/v5` — PostgreSQL client
  - `modernc.org/sqlite` — pure-Go SQLite (no CGO)
  - `github.com/go-chi/chi/v5` — HTTP router
- Run `go mod tidy` on every PR. CI checks (non-mutating — see §12).
- Don't vendor. Don't set `GOFLAGS` globally.

---

## 3. Main entry pattern

Every `cmd/<name>/main.go` follows this skeleton. Copy verbatim, change
the `Use`, `Short`, and registered commands.

```go
// Package main is the entry point for <binary-name>.
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
        // Pre-logger error path — stderr is fine here.
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

// newVersionCmd returns the `version` subcommand.
func newVersionCmd() *cobra.Command {
    return &cobra.Command{
        Use:   "version",
        Short: "Print version information",
        // Run (not RunE) is OK here — version has no failure mode.
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

Note on unused params: Cobra's handler signature includes `*cobra.Command`
and `[]string`. When you don't use one, write `_` instead of the name —
the canonical lint config flags named-but-unused params via
`revive:unused-parameter`.

---

## 4. Cobra patterns

### 4.1 Subcommand factories (shared state)

Each subcommand is built by a `newXxxCmd()` function returning
`*cobra.Command`. Keeps wiring declarative and lets `main.go` read
like a table of contents. For state shared with the parent (config
path, logger), pass by pointer:

```go
// newDaemonCmd returns the `daemon` subcommand.
func newDaemonCmd(configPath *string) *cobra.Command {
    return &cobra.Command{
        Use:   "daemon",
        Short: "Run the daemon",
        RunE: func(cmd *cobra.Command, _ []string) error {
            cfg, err := config.Load(*configPath)
            if err != nil {
                return fmt.Errorf("loading config: %w", err)
            }
            _ = cfg
            // use cmd.Context() for cancellation
            return nil
        },
    }
}
```

### 4.2 Subcommand factories (local flags)

For flags scoped to one subcommand, declare the variable inside the
factory and bind via a closure:

```go
// newGreetCmd returns the `greet --name <name>` subcommand.
func newGreetCmd() *cobra.Command {
    var name string

    cmd := &cobra.Command{
        Use:   "greet",
        Short: "Greet someone",
        RunE: func(_ *cobra.Command, _ []string) error {
            fmt.Printf("Hello, %s!\n", name)
            return nil
        },
    }
    cmd.Flags().StringVar(&name, "name", "world", "who to greet")
    return cmd
}
```

Rules:
- Prefer `RunE` over `Run`. Exception: commands with no failure mode
  (version, help).
- Use `cmd.Context()` for ctx. Never a package-level global.
- For shared state, pass by pointer through factory args.
- For local flags, declare in the factory and bind via closure.
- Global vars in `main.go` are OK *only* for genuinely global things
  (`verbose`, loaded `cfg`, `logger`). Never in business-logic packages.

### 4.3 PersistentFlags + PersistentPreRunE

Put root-level flags on `PersistentFlags`; parse them once in
`PersistentPreRunE`. **Skip setup for commands that don't need it**
(version, help, completion) — otherwise `--version` dies when config
is missing.

```go
// In run():
var (
    configPath string
    verbose    bool
)

rootCmd.PersistentFlags().StringVarP(&configPath, "config", "c",
    config.DefaultConfigPath(), "path to config file")
rootCmd.PersistentFlags().BoolVarP(&verbose, "verbose", "v", false,
    "Enable verbose output")

rootCmd.PersistentPreRunE = func(cmd *cobra.Command, _ []string) error {
    switch cmd.Name() {
    case "version", "help", "completion":
        return nil
    }
    cfg, err := config.Load(configPath)
    if err != nil {
        return fmt.Errorf("loading config: %w", err)
    }
    level := slog.LevelInfo
    if verbose {
        level = slog.LevelDebug
    }
    logger := slog.New(slog.NewTextHandler(os.Stderr,
        &slog.HandlerOptions{Level: level}))
    logger.Info("configured", "machine_id", cfg.MachineID)
    _ = cfg
    return nil
}
```

Gotcha: only the **deepest** `PersistentPreRunE` in a command chain
runs. If a subcommand defines its own `PersistentPreRunE`, the root's
is skipped — your config/logger setup won't happen. Either don't
override on subcommands, or explicitly call the root's
`PersistentPreRunE` from the override.

---

## 5. Config

One package (`internal/config`), one `Load()` function. Sources merge
in order: **defaults → file → env vars** (later overrides earlier).

The full package — runnable as-shown:

```go
// Package config loads and validates application configuration.
package config

import (
    "encoding/json"
    "errors"
    "fmt"
    "os"
    "path/filepath"
    "strconv"
)

// Config is the top-level application configuration.
type Config struct {
    MachineID string         `json:"machine_id"`
    Scope     string         `json:"scope"`
    Postgres  PostgresConfig `json:"postgres"`
}

// PostgresConfig describes a PostgreSQL connection.
type PostgresConfig struct {
    Host     string `json:"host"`
    Port     int    `json:"port"`
    User     string `json:"user"`
    Password string `json:"password"`
    DB       string `json:"db"`
}

// DSN returns the PostgreSQL connection string.
func (p PostgresConfig) DSN() string {
    return fmt.Sprintf("postgres://%s:%s@%s:%d/%s",
        p.User, p.Password, p.Host, p.Port, p.DB)
}

// ErrMissingConfig is returned when a required field is absent.
var ErrMissingConfig = errors.New("missing required config")

// DefaultConfigPath returns the default config file path
// (~/.config/<binary-name>/config.json). Replace <binary-name>.
func DefaultConfigPath() string {
    home, err := os.UserHomeDir()
    if err != nil {
        return "config.json"
    }
    return filepath.Join(home, ".config", "<binary-name>", "config.json")
}

// Load reads config from path, overlays env vars, and validates.
// A missing file is OK (defaults are used); a malformed file is not.
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

func defaults() *Config {
    return &Config{
        Scope: "personal",
        Postgres: PostgresConfig{
            Host: "localhost",
            Port: 5432,
        },
    }
}

// loadFile merges JSON file contents into cfg. Missing files are OK.
func loadFile(cfg *Config, path string) error {
    data, err := os.ReadFile(path) // #nosec G304 -- path is config-sourced
    if errors.Is(err, os.ErrNotExist) {
        return nil
    }
    if err != nil {
        return fmt.Errorf("reading: %w", err)
    }
    if err := json.Unmarshal(data, cfg); err != nil {
        return fmt.Errorf("parsing json: %w", err)
    }
    return nil
}

// loadEnv overlays environment variables using the APP_ prefix.
// Nested structs use underscore separation (e.g. APP_POSTGRES_HOST).
// No reflection — explicit mapping is clearer and easier to debug.
func loadEnv(cfg *Config) {
    if v := os.Getenv("APP_MACHINE_ID"); v != "" {
        cfg.MachineID = v
    }
    if v := os.Getenv("APP_SCOPE"); v != "" {
        cfg.Scope = v
    }
    if v := os.Getenv("APP_POSTGRES_HOST"); v != "" {
        cfg.Postgres.Host = v
    }
    if v := os.Getenv("APP_POSTGRES_PORT"); v != "" {
        if p, err := strconv.Atoi(v); err == nil {
            cfg.Postgres.Port = p
        }
    }
    if v := os.Getenv("APP_POSTGRES_USER"); v != "" {
        cfg.Postgres.User = v
    }
    if v := os.Getenv("APP_POSTGRES_PASSWORD"); v != "" {
        cfg.Postgres.Password = v
    }
    if v := os.Getenv("APP_POSTGRES_DB"); v != "" {
        cfg.Postgres.DB = v
    }
}

// validate returns an error if cfg is incomplete or inconsistent.
func validate(cfg *Config) error {
    if cfg.Postgres.Host == "" {
        return fmt.Errorf("postgres.host: %w", ErrMissingConfig)
    }
    return nil
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
- **No reflection for env-var mapping.** Explicit per-field code is
  more verbose but far easier to debug than a reflection-based layer.
- Config file format is **JSON**. YAML if you have a reason; not by
  default.
- Secrets come from env vars or a secret manager. Never from a config
  file checked into git.
- `APP_` prefix for env vars is convention; replace with project-specific
  prefix (e.g. `WIDGETCTL_`) in real projects.

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
- One logger constructed in `PersistentPreRunE` (for CLIs) or at the
  top of a daemon's run loop.
- **Pass as `*slog.Logger` (pointer).** Stdlib idiom. Don't copy
  loggers by value.
- Don't stash loggers in `context.Context` unless ctx is genuinely
  request-scoped (HTTP handlers).
- Handler choice: `TextHandler` for CLIs and local daemons,
  `JSONHandler` for services shipping logs to a collector. Note that
  the two handlers emit different time formats — switching handler
  types mid-project breaks log ingesters.
- Log keys are lowercase snake_case: `machine_id`, `elapsed_ms`.
- Levels: `Debug` (chatty, off by default), `Info` (normal),
  `Warn` (recoverable), `Error` (action needed). No custom levels.
- Structured attrs over formatted strings. Do:
  `logger.Info("connected", "dsn", dsn)`, not
  `logger.Info(fmt.Sprintf("connected to %s", dsn))`.

---

## 7. Database

### 7.1 PostgreSQL (pgx v5)

Use `pgxpool`. Wrap pool creation to inject timeouts and validate the
connection:

```go
import (
    "context"
    "fmt"
    "time"

    "github.com/jackc/pgx/v5/pgxpool"
)

// Open creates a Postgres connection pool and pings it.
func Open(ctx context.Context, dsn string) (*pgxpool.Pool, error) {
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

Shutdown: `pool.Close()` blocks on in-flight queries. Daemons should
give it a bounded context via a goroutine + select or time out.

### 7.2 SQLite (modernc.org/sqlite, pure Go)

**Put pragmas in the DSN, not in `db.Exec(...)`.** A single `Exec`
pragma only affects the one pooled connection that served it;
subsequent connections revert to defaults and you're back to
"database is locked". DSN pragmas apply to every connection opened
by the pool.

```go
import (
    "database/sql"
    "fmt"

    _ "modernc.org/sqlite" // registers "sqlite" driver
)

// OpenSQLite opens a SQLite database with WAL and busy-timeout pragmas
// set on every pooled connection.
func OpenSQLite(path string) (*sql.DB, error) {
    dsn := fmt.Sprintf(
        "file:%s?_pragma=busy_timeout(5000)&_pragma=journal_mode(wal)&_pragma=foreign_keys(on)",
        path,
    )
    db, err := sql.Open("sqlite", dsn)
    if err != nil {
        return nil, fmt.Errorf("opening sqlite %s: %w", path, err)
    }
    if err := db.Ping(); err != nil {
        _ = db.Close()
        return nil, fmt.Errorf("pinging sqlite %s: %w", path, err)
    }
    return db, nil
}
```

For read-only opens, append `&mode=ro` to the DSN.

Driver name gotcha: `modernc.org/sqlite` registers as `"sqlite"`, not
`"sqlite3"` (which is mattn's CGO driver). `sql.Open("sqlite3", ...)`
with only the modernc import fails with "unknown driver".

---

## 8. HTTP servers

Use `go-chi/chi/v5`. Always set timeouts, always graceful-shutdown.

```go
import (
    "context"
    "errors"
    "fmt"
    "log/slog"
    "net/http"
    "time"
)

// NewServer constructs an http.Server with sane defaults.
func NewServer(addr string, handler http.Handler) *http.Server {
    return &http.Server{
        Addr:              addr,
        Handler:           handler,
        ReadHeaderTimeout: 5 * time.Second,
        ReadTimeout:       30 * time.Second,
        WriteTimeout:      30 * time.Second,
        IdleTimeout:       120 * time.Second,
    }
}

// Run starts srv and blocks until ctx is canceled, then gracefully shuts down.
func Run(ctx context.Context, srv *http.Server, logger *slog.Logger) error {
    errCh := make(chan error, 1)
    go func() {
        if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
            errCh <- err
            return
        }
        errCh <- nil
    }()

    select {
    case <-ctx.Done():
        logger.Info("shutdown requested")
    case err := <-errCh:
        if err != nil {
            return fmt.Errorf("server stopped: %w", err)
        }
        return nil
    }

    shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()
    if err := srv.Shutdown(shutdownCtx); err != nil {
        return fmt.Errorf("shutdown: %w", err)
    }
    return nil
}
```

Handlers take dependencies as struct fields, not globals:

```go
import (
    "log/slog"
    "net/http"

    "github.com/jackc/pgx/v5/pgxpool"
)

// Handler groups HTTP handler dependencies.
type Handler struct {
    DB     *pgxpool.Pool
    Logger *slog.Logger
}

// GetFoo handles GET /foo.
func (h *Handler) GetFoo(w http.ResponseWriter, _ *http.Request) {
    _, _ = w.Write([]byte("ok"))
}
```

**Middleware order matters.** Canonical chain:
`chi.RequestID → chi.RealIP → Logger → Recoverer → Timeout → routes`.

`Recoverer` must go **before** `Timeout`. If a panic happens inside a
timed-out handler and `Recoverer` comes after, the panic isn't caught
and crashes the process.

`Timeout` cancels the request context but does **not** cancel the
handler's goroutine. If the handler does expensive work, check
`r.Context().Done()` yourself.

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
- Stick to the stdlib `testing` package. No third-party assertion
  libraries by default.

---

## 11. Linting — full `.golangci.yml`

Copy this verbatim into `.golangci.yml` at repo root. **Pin
golangci-lint v1.64.x** in CI to match — v2 has a different schema
and this config won't load there.

```yaml
run:
  timeout: 5m
  tests: true

linters:
  enable:
    # Bugs & correctness
    - govet
    - staticcheck       # also covers simple/stylecheck/unused checks
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

    # Command-registration code is inherently long and has duplication
    # across sibling newXxxCmd factories. Anchor regex with ^.
    - path: ^cmd/
      linters:
        - wrapcheck
        - gocognit
        - gocyclo
        - dupl

    # main.go itself is long by nature (command registration)
    - path: ^cmd/.*/main\.go
      linters:
        - funlen

    # Long-running state machines have inherent complexity.
    # Add paths for YOUR daemon/sync dirs as they emerge. Example:
    # - path: ^internal/(sync|extract)/
    #   linters:
    #     - gocognit
    #     - nestif

    # File paths from env/config are expected — gosec G304 noise.
    - linters:
        - gosec
      text: "G304"
```

Notes on the rule set:
- **`staticcheck`** subsumes what `gosimple`, `stylecheck`, and the
  `unused` check used to do separately — don't list the old names.
- **Complexity budgets are tight on purpose.** If you hit them, the
  code usually wants a rewrite, not an exemption. Add per-path
  exemptions only when a specific pattern (e.g. a state-machine loop)
  genuinely can't be simplified.
- **All path exclusions use anchored regex (`^cmd/`)** so
  `internal/foo/cmd/` doesn't accidentally match.
- **`revive:exported`** will flag every exported type/func without a
  doc comment. Comment everything you export — the discipline is worth
  more than the lint bypass.

---

## 12. Build system — full `Makefile`

Copy and replace `<binary-name>` everywhere. The Makefile distinguishes
**mutating** targets (run during local dev — `fmt`, `tidy`) from
**check** targets (run in CI — `fmt-check`, `tidy-check`) so CI
catches drift without rewriting files.

```make
.PHONY: all build
.PHONY: test test-unit test-integration test-all
.PHONY: fmt tidy vet lint security coverage
.PHONY: fmt-check tidy-check
.PHONY: check ci
.PHONY: docker-up docker-down
.PHONY: clean install-local install-local-force dev watch tools help

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
	@printf '$(GREEN)Built: $(BINARY_DIR)/$(BINARY_NAME)$(NC)\n'

# =============================================================================
# Test
# =============================================================================

# -race is always on; catches bugs that only appear under contention.
test-unit:
	@printf '$(GREEN)Running unit tests...$(NC)\n'
	go test -race -v ./...

test: test-unit

# Requires Docker + tests/docker-compose.yml; skip unless you have
# integration tests. Runs cleanup even on failure.
test-integration: docker-up
	@printf '$(GREEN)Running integration tests...$(NC)\n'
	@set -e; rc=0; \
		(cd tests && docker compose run --rm test-runner \
			go test -v -tags=integration ./integration/...) || rc=$$?; \
		$(MAKE) docker-down; \
		exit $$rc

test-all: test-unit test-integration
	@printf '$(GREEN)All tests passed!$(NC)\n'

# =============================================================================
# Code quality — mutating (local dev)
# =============================================================================

fmt:
	@printf '$(GREEN)Formatting code...$(NC)\n'
	@if ! command -v goimports >/dev/null 2>&1; then \
		printf '$(RED)goimports not installed. Run make tools.$(NC)\n'; exit 1; \
	fi
	go fmt ./...
	goimports -w .

tidy:
	go mod tidy

# =============================================================================
# Code quality — non-mutating (CI)
# =============================================================================

# Fails if any files need formatting. Lists offenders on stderr.
fmt-check:
	@printf '$(GREEN)Checking formatting...$(NC)\n'
	@out=$$(gofmt -l .); \
		if [ -n "$$out" ]; then \
			printf '$(RED)Files need gofmt:$(NC)\n%s\n' "$$out"; exit 1; \
		fi
	@if command -v goimports >/dev/null 2>&1; then \
		out=$$(goimports -l .); \
		if [ -n "$$out" ]; then \
			printf '$(RED)Files need goimports:$(NC)\n%s\n' "$$out"; exit 1; \
		fi; \
	fi

# Fails if go.mod/go.sum would change after tidy.
tidy-check:
	@printf '$(GREEN)Checking go.mod/go.sum...$(NC)\n'
	@go mod tidy
	@if ! git diff --quiet -- go.mod go.sum 2>/dev/null; then \
		printf '$(RED)go.mod or go.sum changed after tidy; run make tidy and commit.$(NC)\n'; \
		git diff -- go.mod go.sum; exit 1; \
	fi

# =============================================================================
# Static analysis (non-mutating)
# =============================================================================

vet:
	@printf '$(GREEN)Running go vet...$(NC)\n'
	go vet ./...

lint:
	@if ! command -v golangci-lint >/dev/null 2>&1; then \
		printf '$(RED)golangci-lint not installed. Run make tools.$(NC)\n'; exit 1; \
	fi
	@printf '$(GREEN)Running golangci-lint...$(NC)\n'
	golangci-lint run ./...

security:
	@if ! command -v gosec >/dev/null 2>&1; then \
		printf '$(RED)gosec not installed. Run make tools.$(NC)\n'; exit 1; \
	fi
	@printf '$(GREEN)Running gosec...$(NC)\n'
	gosec -quiet ./...

coverage:
	@printf '$(GREEN)Running tests with coverage...$(NC)\n'
	go test -race -coverprofile=coverage.out -covermode=atomic ./...
	@printf '\n$(GREEN)Coverage summary:$(NC)\n'
	@go tool cover -func=coverage.out | grep -E "^total:|internal/"
	@go tool cover -html=coverage.out -o coverage.html
	@printf '$(GREEN)Full report: coverage.html$(NC)\n'

# =============================================================================
# CI pipelines
# =============================================================================

# `check` is the fast local pipeline. Mutates files (fmt, tidy). Use
# during active development.
check: fmt tidy vet test-unit build
	@printf '$(GREEN)Quick check passed!$(NC)\n'

# `ci` is the non-mutating pipeline. Use in CI and pre-push hooks.
# Catches formatting/tidy drift that `check` would silently fix.
ci: fmt-check tidy-check vet lint security test-unit coverage build
	@printf '\n$(GREEN)========================================$(NC)\n'
	@printf '$(GREEN)  CI Pipeline completed successfully!  $(NC)\n'
	@printf '$(GREEN)========================================$(NC)\n'

# =============================================================================
# Docker (only if tests/docker-compose.yml exists)
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

# Interactive install. Falls back to force-install under non-TTY.
install-local: build
	@mkdir -p $$HOME/.local/bin
	@if [ ! -t 0 ]; then \
		cp $(BINARY_DIR)/$(BINARY_NAME) $$HOME/.local/bin/; \
		printf '$(GREEN)Installed to ~/.local/bin/$(BINARY_NAME) (non-interactive).$(NC)\n'; \
		exit 0; \
	fi; \
	if [ -x $$HOME/.local/bin/$(BINARY_NAME) ]; then \
		printf '$(YELLOW)Current:$(NC) %s\n' "$$($$HOME/.local/bin/$(BINARY_NAME) version 2>/dev/null || echo unknown)"; \
		printf '$(GREEN)New:$(NC)     $(BINARY_NAME) version $(VERSION) (built $(BUILD_TIME))\n'; \
		read -p "Replace existing installation? [y/N] " confirm; \
		if [ "$$confirm" != "y" ] && [ "$$confirm" != "Y" ]; then \
			printf '$(YELLOW)Cancelled$(NC)\n'; exit 1; \
		fi; \
	else \
		printf '$(GREEN)Installing:$(NC) $(BINARY_NAME) version $(VERSION) (built $(BUILD_TIME))\n'; \
	fi; \
	cp $(BINARY_DIR)/$(BINARY_NAME) $$HOME/.local/bin/; \
	printf '$(GREEN)Installed to ~/.local/bin/$(BINARY_NAME)$(NC)\n'

install-local-force: build
	@mkdir -p $$HOME/.local/bin
	cp $(BINARY_DIR)/$(BINARY_NAME) $$HOME/.local/bin/
	@printf '$(GREEN)Installed to ~/.local/bin/$(BINARY_NAME)$(NC)\n'

dev: build
	./$(BINARY_DIR)/$(BINARY_NAME)

# Rebuild on change. Requires entr (https://eradman.com/entrproject/).
watch:
	@if ! command -v entr >/dev/null 2>&1; then \
		printf '$(RED)entr not installed. See https://eradman.com/entrproject/.$(NC)\n'; exit 1; \
	fi
	find . -name '*.go' | entr -c make build

tools:
	@printf '$(GREEN)Installing development tools...$(NC)\n'
	go install golang.org/x/tools/cmd/goimports@latest
	go install github.com/golangci/golangci-lint/cmd/golangci-lint@v1.64.8
	go install github.com/securego/gosec/v2/cmd/gosec@latest

help:
	@echo "Available targets:"
	@echo "  build               Build the binary"
	@echo "  install-local       Build and install to ~/.local/bin (TTY: confirm; non-TTY: force)"
	@echo "  install-local-force Install without confirmation"
	@echo "  test                Run unit tests (-race)"
	@echo "  test-unit           Run unit tests (-race)"
	@echo "  test-integration    Run integration tests (requires Docker)"
	@echo "  coverage            Generate coverage report"
	@echo "  fmt                 Format code (mutating)"
	@echo "  tidy                go mod tidy (mutating)"
	@echo "  fmt-check           Check formatting (non-mutating)"
	@echo "  tidy-check          Check go.mod/go.sum (non-mutating)"
	@echo "  vet                 go vet"
	@echo "  lint                golangci-lint"
	@echo "  security            gosec"
	@echo "  check               fmt + tidy + vet + test + build (local, mutates)"
	@echo "  ci                  Full non-mutating CI pipeline"
	@echo "  dev                 Build and run"
	@echo "  watch               Rebuild on change (requires entr)"
	@echo "  clean               Remove build artifacts"
	@echo "  tools               Install dev tools (goimports, golangci-lint@v1.64.8, gosec)"
	@echo "  help                Show this help"
```

**Ldflags for version injection are mandatory.** `<binary> version`
should always print the real commit + build timestamp. Note: `git describe`
silently falls back to `"dev"` if run outside a git repo — always run
`git init` as the first checklist step.

**`~/.local/bin` must be on `$PATH`** for `install-local` to be useful.
Typical shell setup:
```sh
export PATH="$HOME/.local/bin:$PATH"
```

---

## 13. CI — full `.github/workflows/ci.yml`

Every CI job maps 1:1 to a `make` target, so the Makefile is the single
source of truth. CI and local behavior can't drift.

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  GO_VERSION: '1.25'
  GOLANGCI_LINT_VERSION: 'v1.64.8'

jobs:
  fmt:
    name: Format check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: ${{ env.GO_VERSION }}
          cache: true
      - name: Install goimports
        run: go install golang.org/x/tools/cmd/goimports@latest
      - name: Check formatting
        run: make fmt-check

  tidy:
    name: Tidy check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: ${{ env.GO_VERSION }}
          cache: true
      - name: Check go.mod / go.sum
        run: make tidy-check

  vet:
    name: Vet
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: ${{ env.GO_VERSION }}
          cache: true
      - run: make vet

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
          version: ${{ env.GOLANGCI_LINT_VERSION }}
          args: --timeout=5m

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
      - run: make security

  test:
    name: Test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: ${{ env.GO_VERSION }}
          cache: true
      - run: make test-unit

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
        run: make coverage
      # Codecov upload is optional; requires CODECOV_TOKEN secret for
      # private repos. Skipped when no token is set.
      - name: Upload coverage to Codecov
        if: ${{ secrets.CODECOV_TOKEN != '' }}
        uses: codecov/codecov-action@v4
        with:
          files: ./coverage.out
          fail_ci_if_error: false
          token: ${{ secrets.CODECOV_TOKEN }}

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
      # CGO off for cross-compile — modernc.org/sqlite is pure Go and
      # no other dep needs CGO. If you add a CGO dep, handle per-arch
      # toolchains explicitly instead of flipping this on.
      - name: Build
        env:
          GOOS: linux
          GOARCH: ${{ matrix.goarch }}
          CGO_ENABLED: 0
        run: |
          mkdir -p dist
          go build -ldflags "-X main.Version=${{ github.sha }} -X main.BuildTime=$(date -u '+%Y-%m-%dT%H:%M:%SZ')" \
            -o dist/<binary-name>-linux-${{ matrix.goarch }} ./cmd/<binary-name>
      - uses: actions/upload-artifact@v4
        with:
          name: <binary-name>-linux-${{ matrix.goarch }}
          path: dist/<binary-name>-linux-${{ matrix.goarch }}
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
      - run: make test-unit

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
          CGO_ENABLED: 0
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

For daemons that should run as a systemd user unit, include an install
target in the Makefile that drops a unit file into
`~/.config/systemd/user/`. Minimal template:

```ini
[Unit]
Description=<one-line description>
After=<dependency.service>

[Service]
Type=simple
ExecStart=%h/.local/bin/<binary-name> daemon
Restart=on-failure
RestartSec=10
Environment=HOME=%h

[Install]
WantedBy=default.target
```

Install via `systemctl --user daemon-reload && systemctl --user enable --now <binary-name>.service`.

**`Type=simple` on purpose.** `Type=notify` requires the Go program to
call `sd_notify(READY=1)` via `github.com/coreos/go-systemd/v22/daemon`;
without that, the service hangs in `activating` and systemd kills it.
If you genuinely need `notify` + `WatchdogSec`, add the go-systemd
dependency and the notify calls, then flip the unit type.

**User units only.** Don't install system-level services.

---

## 15. Gotchas — stuff we've been burned by

- **Cobra `PersistentPreRunE` runs for *everything*, including `--help`
  and `--version`.** If it unconditionally loads config, `--version`
  dies on a machine with no config. Always early-return on
  `version` / `help` / `completion`.
- **Cobra `PersistentPreRunE` inheritance: only the deepest override
  in a command chain runs.** If a subcommand defines its own, the
  root's is skipped. Your config/logger setup silently doesn't happen.
- **SQLite pragmas via `db.Exec("PRAGMA ...")` only affect one pooled
  connection.** Other connections revert to defaults → `database is
  locked` under contention. Put pragmas in the DSN (`?_pragma=...`)
  so every connection gets them.
- **`modernc.org/sqlite` registers as `"sqlite"`**, not `"sqlite3"`
  (which is mattn/CGO). `sql.Open("sqlite3", ...)` with only
  `_ "modernc.org/sqlite"` fails "unknown driver".
- **`modernc.org/sqlite` does NOT enable `PRAGMA foreign_keys=ON` by
  default.** Add `&_pragma=foreign_keys(on)` to the DSN if you rely on
  FK enforcement.
- **`pgxpool.Pool.Close()` blocks on in-flight queries.** Daemons
  need a time-bounded shutdown (goroutine + `time.After` race).
- **`pgxpool.ParseConfig` silently ignores unknown DSN parameters.**
  Validate the fields of the returned config, or typo'd options will
  be invisible.
- **`wrapcheck` false positives** when wrapping errors from your own
  helpers. Either add to `ignoreSigs` or return pre-wrapped.
- **`httptest.NewServer` vs `NewTLSServer`** — if the client is
  hard-coded to HTTPS, the plain version gives an unhelpful
  "unknown protocol" error. Use the TLS variant.
- **`go-chi`'s `Timeout` middleware doesn't cancel the handler's
  goroutine** — it only cancels the request context. If you have
  expensive work, check `r.Context().Done()` yourself.
- **`middleware.Recoverer` must come before `middleware.Timeout`.**
  A panic inside a timed-out handler otherwise escapes Recoverer's
  catch and crashes the process.
- **`slog.TextHandler` and `JSONHandler` emit different time formats.**
  Switching handler types mid-project breaks log ingesters that expect
  a fixed shape.
- **`signal.NotifyContext` only covers the current process.** If you
  `os/exec` a subprocess, propagate the context via
  `exec.CommandContext` or the child won't die on Ctrl-C.
- **`modernc.org/sqlite` is pure Go but slower than CGO alternatives.**
  For hot paths (embedding stores, high-QPS), profile before assuming
  it's fine.
- **`go build` without ldflags** produces a binary that reports
  `version dev (built unknown)`. Always build through the Makefile in
  local dev, or set `LDFLAGS` explicitly.
- **`git describe` silently falls back to `"dev"` outside a git repo.**
  Always `git init` first so your version string reflects reality.
- **`go test` without `-race`** passes locally but can fail in CI.
  The Makefile always passes `-race`; don't skip it.
- **`t.Parallel()` + shared state** (mutated globals, shared DB)
  flakes nondeterministically. Either isolate state per test or don't
  parallelize that test.

---

## 16. Checklist for a new project

Work through in order. Each step has a template earlier in this doc.

- [ ] `mkdir <project> && cd <project>`
- [ ] `git init -b main`
- [ ] `go mod init github.com/<user>/<project>`
- [ ] Open `go.mod` and confirm/edit the `go` directive to `go 1.25`
      (matches `GO_VERSION` in CI)
- [ ] `mkdir -p cmd/<binary-name> internal/config .github/workflows`
- [ ] `cmd/<binary-name>/main.go` from §3 skeleton (optionally with
      the §4.2 subcommand example)
- [ ] `internal/config/config.go` from §5 (replace `APP_` env prefix
      and `<binary-name>` placeholder)
- [ ] `.golangci.yml` from §11 verbatim
- [ ] `Makefile` from §12, replace `<binary-name>`
- [ ] `.github/workflows/ci.yml` from §13, replace `<binary-name>`
- [ ] `.github/workflows/release.yml` if you plan to cut tagged releases
- [ ] `.gitignore`:
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
- [ ] `README.md` — at minimum: what it does, install, usage, config,
      dev. A one-page file is fine; long-form docs can live elsewhere.
- [ ] `LICENSE` — pick one; MIT is the default for personal projects.
- [ ] `go get github.com/spf13/cobra` (and any other deps you reference
      in the skeleton)
- [ ] `go mod tidy`
- [ ] Write at least one table-driven test (see §10). `make ci` runs
      coverage; an empty test suite gives false reassurance.
- [ ] Ensure `~/.local/bin` is on `$PATH` if you want `make install-local`
- [ ] `make tools` to install dev tools (goimports, golangci-lint@v1.64.8, gosec)
- [ ] `git add . && git commit -m "initial commit"` — needed so
      `git describe` produces something other than the `"dev"` fallback
      for subsequent builds.
- [ ] `make ci` should pass cleanly before push.

---

## 17. What's intentionally NOT here

- **Observability / metrics.** No canonical Prometheus pattern yet.
  Add when one exists.
- **Secrets management.** Varies by environment (1Password CLI, Vault,
  AWS SM, plain env). Pick per-project; don't put secrets in config
  files regardless.
- **Frontend / UI.** Out of scope.
- **SQL migrations.** No preferred tool yet (golang-migrate vs sqlc vs
  atlas). Pick per-project, document in the project's README.
- **Dockerfile.** Write one when you need it; no canonical template
  worth enforcing.
- **Project-scaffolding script.** Copy-paste from this doc is the
  canonical workflow. A cookiecutter is on the wishlist.
