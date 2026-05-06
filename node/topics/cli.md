# Topic: CLI

> **Status:** stub. Will harden against a real CLI project.

## Recommended baseline (subject to real-world tightening)

| Concern | Choice |
|---|---|
| Argument parser | `commander` (or `yargs` for very complex apps) |
| Bin entry | Single binary, `cmd/<name>/index.ts` thin wiring |
| Config | Layered: defaults → file → env → flags |
| Logging | `pino` (structured) or just `console` for trivial CLIs |
| Output | `picocolors` for terminal colors (if needed); detect TTY before colorizing |
| Distribution | npm package with `bin` field, or compiled to a single binary via `pkg` / `vercel/pkg` / native compile |

## Notes (to expand)

- The Go pattern (thin `main.go` + factory subcommands) translates 1:1 to TS with `commander`.
- For "global install via npm" tools, the `bin` field is the contract.
- For "single-binary distribution" tools, evaluate `bun build --compile` before reaching for `pkg`.

## Open questions

- Compile-to-binary toolchain. (`bun build --compile` is appealing but ties you to Bun runtime.)
- Whether to standardize on a config file format (JSON / YAML / TOML).
