# LESSONS.md — hard-won gotchas from the source project

Portable failure modes discovered the expensive way. Each one cost a real
debugging session; reading this list is cheaper. Grouped by area.

## Git / worktree mechanics

- **ff-merge from the main checkout, not the worktree.** Running
  `git merge --ff-only bead/<id>` *inside* the bead worktree reports
  "Already up to date" and silently no-ops — the branch is merging into
  itself. The join always runs from the main checkout.
- **`git worktree add` paths are cwd-relative.** Running it after `cd` into a
  subdir lands the worktree at the wrong parent. Always run from the repo
  root of the main checkout.
- **Spike branches are permanent.** `spike/*` refs are the project's
  experiment record — never `git branch -d`, even after ff-merge. Close-out =
  worktree remove + `bd close` only. `branch -d` applies to `bead/*` only.
- **Never on `main`:** `--amend`, force-push, `git add -A` / `git add .`.
  Explicit-path staging everywhere; `--amend` only on a private bead branch.

## Gates / verification

- **Typecheck with the real build, not a bare `tsc --noEmit`.** In a
  project-references / multi-tsconfig setup, `tsc --noEmit` at the wrong level
  checks nothing. Use `tsc -b` or the actual build command as the gate.
- **Gate parity: builders run the same gates the pre-commit hook enforces.**
  If husky runs lint + typecheck, a builder that only ran tests will fail at
  commit time. State the full gate list in every builder prompt.
- **Run the test suite after copy/label/i18n changes, not just the build.**
  DOM tests query rendered text; "it's only copy" still breaks them.
- **Gates green ≠ safe to merge. Review the diff-stat file list first.**
  Builders have deleted unrelated live features that were dead code *to
  them*; tests still passed. Scope check (`git diff --stat main`) is part of
  every review, before reading a single hunk.
- **Verify a fix by exercising the flow, not by re-reading the code.** If
  checking requires running the app, spawn an agent to run it and report.
- **Defer visual QA when working autonomously.** User away ⇒ ship
  gate-verifiable work; anything needing an eye-check (canvas/pixel/layout)
  gets a tracked follow-up bead marked VISUAL QA instead of a guess.

## Agents / delegation

- **Pin the model on every spawn.** Subagents inherit the parent's model when
  `model` is omitted — a "cheap mechanical edit" silently runs on the most
  expensive model. Always pass `model` and put it in the agent's name
  (`<bead>-<short>-sonnet`).
- **Recursion guard is structural.** A `general-purpose` agent holds the
  Agent tool and will re-delegate leaf work under `auto`, spawning unnamed
  orphans no shutdown handle reaches. Give leaf work to agents whose tool
  grant lacks Agent. Prompt pleas ("do not spawn") don't work; tool grants do.
- **Reuse a warm builder for batched small edits.** The waste in
  spawn-per-edit is the cold start (worktree deps install + gate run), not
  the reaping. Batch related small edits into one builder session.
- **Recall incomplete agents; don't silently finish their work.** Partial
  result ⇒ `SendMessage` the same agent with exactly what's missing. Keeps
  the audit trail honest and the orchestrator's hands off the tree.
- **Fat work orders beat mid-flight clarification.** Freezing
  `--design`/`--acceptance` into the bead at creation makes the spawn
  near-mechanical and cuts orchestrator thinking per bead.

## bd (beads)

- **Pass rich fields via file, not inline.** Backticks/parens in
  `--design`/`--acceptance` get eaten by zsh. Write the text to a scratch
  file and pass `--design "$(cat file)"`.
- **`.beads/issues.jsonl` is a passive export.** Never hand-merge it;
  regenerate. Issues live in the Dolt DB; row-level merge makes concurrent
  bd writes across sessions safe.
- **Use `bd remember` for cross-session insights** (agent quirks, environment
  facts) — searchable via `bd memories <keyword>`.

## Process / product

- **Two-session concurrency needs disjoint write surfaces.** Two tabs editing
  the same tree clobbered each other's work — the origin of the two-tab
  model. If you add a second session, partition what it may write.
- **Spike before build.** Any feature resting on an unproven assumption gets
  a throwaway spike with a written GO/PARTIAL/NO-GO verdict first. Verdicts
  live in `docs/spike-*.md`; several planned features died cheaply there.
- **A build-time flag may encode a legal/launch gate.** When migrating a
  compile-time flag to runtime/remote config, keep the hardcoded guard AND-ed
  in — the runtime path can be flipped from a console and silently opens the
  hatch the flag existed to keep shut.
- **Cache keys carry every parameter that affects the cached value.**
  A render/LRU key missing one dimension (scale, rotation, mode) serves a
  stale artifact that misaligns everything downstream. When adding a render
  param, grep for the cache key first.
- **Extract-then-test is always available.** "The buggy logic isn't unit
  testable" means it's buried in a component/effect — extract the decision
  into a pure function, test that, wire it back. No fix ships without its
  regression test.
- **Every solved bug is logged the day it's solved** (PROBLEMS dual-file).
  The index is the project's immune system — half the entries prevented a
  second occurrence.
