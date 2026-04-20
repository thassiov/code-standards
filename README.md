# code-standards

Personal, opinionated coding standards for projects I own or maintain.
Self-contained — every template and config you need is embedded in the
relevant standards doc. Copy-paste is the intended workflow.

## Languages

| Language | Doc | Covers |
|----------|-----|--------|
| Go | [go.md](go.md) | CLIs, daemons, HTTP services. Cobra, slog, pgx, SQLite, go-chi. Full `.golangci.yml`, `Makefile`, and GitHub Actions templates. |

More to be added as they prove themselves in real projects.

## Process

- [testing-standards.md](testing-standards.md) — how each language doc
  in this repo is stress-tested (scaffold + adversarial review +
  regression verification) before being promoted to "the standard."
  Run this on any new language doc before publishing.

## How to use

1. Start a new project from scratch? Work through the checklist at the
   end of the relevant language doc.
2. Reviewing an existing project? Diff it section-by-section against
   the doc and open issues for gaps.
3. Adding a new language doc? Apply the process in
   [testing-standards.md](testing-standards.md) before merging.
4. Disagree with something? Open a PR — these are living documents.

## Principles

- **Self-contained.** No "copy from that other repo" references. Every
  template lives in the doc.
- **Rigorous, not elaborate.** Prefer one enforced rule over three
  optional suggestions. Prefer code over prose when both convey the same
  thing.
- **Grounded in what actually works.** Every pattern in here has been
  burned into a real project. Nothing speculative.
- **Explicit about what's not covered.** Each doc ends with a "what's
  intentionally NOT here" section so nobody mistakes a gap for a
  recommendation.

## License

MIT — see [LICENSE](LICENSE).
