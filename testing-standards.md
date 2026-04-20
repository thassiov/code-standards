# Testing a Standards Doc

How to verify a standards doc actually holds up before shipping it.
This is the process used to harden [go.md](go.md), and it's the
default approach for any new language doc added to this repo.

**Scope:** standards documents in this repo (Go, future TS/Python/Rust,
etc.). Not a general-purpose doc-testing methodology.

## When to run this

- Before publishing a new standards doc for the first time.
- After any major revision that changes embedded templates (Makefile,
  CI, lint config, scaffolding checklist).
- Before promoting a doc from "draft" to "the standard."

Skip for pure copy-edits, typo fixes, or prose-only changes.

## The method

Two tests run **in parallel** against the draft, then results are
consolidated into a prioritized punch list. Apply fixes, then run
one verification pass.

```
                    ┌──────────────────────┐
draft doc  ───►    │ 1. Scaffold test     │ ────┐
         \         └──────────────────────┘     │
          \                                     ├──►  consolidated
           \        ┌──────────────────────┐    │     punch list
            └──►   │ 2. Adversarial review │ ───┘
                    └──────────────────────┘

                    ┌──────────────────────┐
fixed doc  ───►    │ 3. Re-scaffold       │ ────►  verification
                    │    + regression     │
                    │    injection        │
                    └──────────────────────┘
```

The scaffold test and the adversarial review **must be independent**.
Run them as separate agents (different context, no shared findings)
so overlap is genuine convergence, not echo.

---

## 1. Scaffold test

**Goal:** build a tiny but realistic project using ONLY the doc. If
the checklist can't be completed without outside knowledge, or if any
step produces ambiguity, that's a finding.

**Rules:**
- The tester reads only the doc under test. No peeking at existing
  projects that inspired it.
- Follow the checklist literally, in order.
- Treat every gap as a finding — don't paper over, don't infer.
- Build something non-trivial enough to exercise the major patterns:
  subcommand factories, config loading, at least one unit test.
- Run the doc's own CI pipeline (`make ci`, `npm ci`, `cargo ci`, etc.).
  Record every failure, every silent pass, every "how did that work?"

**What to scaffold** (Go example — adapt for other languages):

| Exercises | Build |
|-----------|-------|
| Entry-point skeleton | A `version` subcommand |
| Flag parsing + factory pattern | A `greet --name <name>` subcommand |
| Config loading | An `internal/config` package reading one env var |
| Testing pattern | At least one table-driven unit test |
| Linter config | Code that would trip the canonical rules if miswritten |

**Regression injection (after fixes):**
After applying fixes from round 1, run the scaffold test **again** with
these additions:

1. Introduce a lint error (e.g. an unused variable). Confirm `make ci`
   exits non-zero with the actual error on stderr.
2. Introduce a formatting drift (trailing whitespace). Confirm the
   formatting check catches it and names the file.
3. Introduce a dirty-tree `go.mod` or `package.json`. Confirm the tidy
   check fails.
4. Fix all three. Confirm `make ci` passes clean.

If any of these four don't behave, the pipeline has a silent-failure
bug — the doc's quality gates are decorative, not load-bearing.

---

## 2. Adversarial review

**Goal:** read the doc hunting for problems. No summaries, no
endorsements. A correct section produces no output; only holes are
reported.

**Categories to hunt:**

1. **Broken examples** — do code snippets compile? Do imports line up
   with what's used? Are referenced functions/types defined anywhere?
2. **Contradictions** — does §X say something §Y enforces against?
   (E.g. "pass by value" in one section, pointer in another; example
   code that fails the enforced lint config.)
3. **Hidden assumptions** — steps that assume tools, env vars, or
   state the doc never establishes.
4. **Missing checklist steps** — items required to reach working state
   that aren't in the "new project" flow.
5. **Stale / wrong advice** — deprecated APIs, linters folded into
   others, outdated versions, renamed flags.
6. **Internal consistency** — Makefile vs CI config; `.PHONY` list
   completeness; `help` target coverage; every referenced binary
   installable from `tools` target.
7. **Lint/format config correctness** — invalid rule names, settings
   that don't apply to the pinned version, contradictory exclude
   rules, regex anchoring.
8. **Unlisted gotchas** — any pitfall you know about that §15 (or
   equivalent) doesn't flag.

**What not to do:**
- Don't report "I'd prefer X" opinions. Only concrete errors.
- Don't skim and summarize. Line-by-line, section-by-section.
- Don't propose large rewrites — suggest the minimal fix.

---

## 3. Consolidated punch list

Merge the two reports into one file with 5 tiers:

| Tier | Meaning |
|------|---------|
| **1 — critical** | Blocks adoption. Silent failures, uncompilable examples, broken quality gates. |
| **2 — important** | Serious gaps that trip users in practice (missing checklist steps, drifting CI vs local). |
| **3 — consistency** | Contradictions and style inconsistencies that would cause confusion. |
| **4 — gotchas** | Known-issue additions to the doc's gotcha list. |
| **5 — nits** | Small cleanups. Optional. |

**Consensus findings** (both methods flagged the same issue
independently) are always Tier 1 or 2 — high confidence, fix first.

**Yield observation** from practice: adversarial review typically
produces 2–3× more findings than the scaffold. But the scaffold
catches silent-failure patterns that pure reading misses (exit codes,
Makefile shell-chain bugs). Both are non-optional.

---

## 4. Fix, then re-verify

After applying fixes:

1. Commit the fixes with a message listing the findings addressed.
2. Re-run the scaffold test with regression injection (step 1 above).
3. If new issues surface (and they will — some only appear now that
   gates actually run), apply a second pass.
4. Only once regression signals behave honestly on a clean run, ship.

Typical number of passes: **2**. Round 1 fixes the major bugs. Round 2
catches issues that couldn't surface when round 1's pipeline was
silently passing.

---

## 5. Agent prompts

Copy-paste-ready. Replace `<DOC PATH>` and the language-specific bits.

### 5.1 Scaffold test prompt

```
**Task**: Stress-test a standards document by following its "new
project" checklist literally, as if you're a new engineer who only
has the doc. Find gaps, ambiguities, and errors.

**Doc to test**: <DOC PATH>

**Hard rule**: Use ONLY the contents of that doc. Do NOT read any
other project for hints. If the doc doesn't tell you how to do
something, treat that as a gap.

**Environment notes**: <language runtime version>, relevant tools
that are or are not pre-installed. Note any version drift.

**What to scaffold**: A small project called <NAME> with:
- Entry-point subcommand that exercises the skeleton
- At least one subcommand with a local flag (exercises flag pattern)
- A config package exercising the canonical Load() pattern
- At least one unit test using the canonical test pattern

**What to run, in order**:
1. The doc's "install tools" step
2. The doc's "fast local check" pipeline
3. The doc's "full CI" pipeline

Record every failure, lint warning, ambiguity, and place where you
had to guess.

**Deliverable**: A punch list. For each finding:
- Severity: blocker / gap / nit
- Section of the doc it relates to
- What the doc says vs. what actually happened
- Suggested fix in one sentence

Do NOT edit the doc. The scaffold is throwaway. Report in under 500
words.
```

### 5.2 Adversarial review prompt

```
**Task**: Adversarial review of a standards document. Hunt for
problems — don't summarize or endorse. If something is correct, skip
it entirely.

**Doc**: <DOC PATH>

**Hunt for**:
1. Broken examples (do snippets compile? imports line up?)
2. Contradictions between sections
3. Hidden assumptions (tools, env, state)
4. Missing checklist steps
5. Stale or wrong advice (deprecated APIs, renamed tools)
6. Makefile / build-script internal consistency
7. Lint / format config correctness
8. Unlisted gotchas (add what's missing)
9. Consistency within the doc (does §X's pattern match §Y's enforcement?)

**Do NOT check**: CI/CD platform specifics. Style preferences.

**Deliverable**: Punch list in response text. For each finding:
- Severity: blocker / gap / nit
- Location (section + short quote)
- The problem, stated concretely
- Suggested fix in one sentence

Skip anything correct. No endorsements. Report in under 500 words.
```

### 5.3 Regression-verification prompt (for round 2+)

```
**Task**: Re-test a REVISED standards document. Verify that previously
identified fixes hold, and that the pipeline fails honestly when
things are broken.

**Doc**: <DOC PATH>
**Previous findings (for regression awareness)**: <bullet list>

Scaffold the same project as before. Then:
1. Complete the checklist; confirm `make ci` passes clean.
2. Introduce a lint error. Confirm `make ci` fails non-zero.
3. Introduce a formatting drift. Confirm the fmt check catches it.
4. Introduce a dirty go.mod / lockfile. Confirm the tidy check catches it.
5. Fix all three; confirm `make ci` passes again.

**Deliverable**: For each previous finding, state FIXED / NOT FIXED /
PARTIAL. Then list any NEW issues. Include an "honest signal" section:
did the pipeline actually fail when it should have?

Report in under 500 words.
```

---

## 6. Gotchas from practice

Things that went wrong when this process was applied to `go.md`:

- **The subagent harness may block writes to the workbench path.**
  Tell the agent to return the punch list inline in response text, not
  write it to a file. The orchestrator saves the file after.
- **"Silent green" is the most common failure mode.** Any `||` fallback
  in a Makefile recipe is suspect — trace its exit codes explicitly.
  Assume the gate doesn't work until you've seen it fail on purpose.
- **Round 2 surfaces issues round 1 hid.** When a gate starts working
  (e.g. `make security` actually runs gosec), it surfaces problems
  that were invisible when the gate was a no-op. Always plan for at
  least two passes.
- **Prompt length matters.** Both agents got more useful when the
  prompt stated HARD RULES ("use ONLY the doc") and a length cap
  ("under 500 words"). Shorter prompts produced shallower findings.
- **Explicit category lists for the adversarial reviewer.** Without
  them, the agent drifts into style opinions. The 9-category hunt
  list (above) keeps it focused on concrete defects.
- **Cross-agent agreement is load-bearing.** When both the scaffold
  and the review independently flag the same issue, it's always
  Tier 1. Don't second-guess consensus.
- **Scaffold artifacts are not the product.** Delete the scaffold
  between rounds (`rm -rf test-scaffold/`) to ensure round 2 starts
  clean — otherwise residual state from round 1's partial fixes
  poisons the result.
