# CLAUDE.md

## Project
{{PROJECT_ONELINER — what the app is, stack, who uses it}}

## Repo layout
{{REPO_LAYOUT — one bullet per top-level dir; state where dev commands run, e.g.:}}
- `frontend/app/` — main source; dev commands: `cd frontend/app && {{DEV_CMDS: pnpm run dev|build|test}}`
- `backend/` — {{...}}
- Repo root — `docs/`, `plans/`, `spike/`, `problems/`, meta `*.md` files (this file, ROADMAP.md, WORKFLOW.md), tooling dirs (`.beads/`, `.claude/`, `.husky/`)

## Working process
Full playbook in [`WORKFLOW.md`](./WORKFLOW.md). Hard-won gotchas in [`LESSONS.md`](./LESSONS.md).

> ### 🚨 BEFORE ANY `src/` CHANGE — STOP, PICK A LANE 🚨
> Do not write code until all three hold. This gate is skipped too often — treat a miss as a defect.
>
> **Two invariants — BOTH lanes, no exceptions:**
> 1. **No bead, no code.** Every code change starts with `bd create` (+ `bd update <id> --claim` before the first edit). A change with no bead is out of process — stop and file one.
> 2. **Orchestrator never edits `src/`.** Every source change is a subagent deliverable. The orchestrator only: plans, spawns, reviews, runs gates, commits/merges. (Meta docs — this file, ROADMAP/WORKFLOW/plans — are planning-lane work the orchestrator *may* edit directly.)
>
> **Then pick the lane by blast radius:**
>
> | | **LIGHT lane** | **FULL lane** |
> |---|---|---|
> | **Use when** | rename / label / copy / doc, single one-liner, **no behavior change** | feature, bugfix, multi-file, any logic/behavior change, anything risky |
> | **bd** | `bd create` (track only) | `bd create` + `--claim` |
> | **Branch** | current branch OK | `git worktree add ../{{REPO}}-wt/<id> -b bead/<id> main` (one bead = one worktree = one branch) |
> | **Edits** | delegate to builder | delegate to builder(s) |
> | **Test** | n/a (no behavior change) | **every bugfix ships a regression test, same commit** |
> | **Land** | commit on the branch | gates green → `git merge --ff-only` into main checkout |
>
> **When unsure which lane → use FULL.** "It's just a small change" is how LIGHT gets abused — behavior change or multi-file is always FULL, regardless of line count.

TLDR of the rest:

- **Lifecycle:** roadmap `#N` → `plans/*.md` → **spike-gate risky assumptions** (`docs/spike-*.md`, GO/PARTIAL/NO-GO) → build one-commit-per-step (tick ROADMAP checkbox same commit) → log fixes in `problems/PROBLEMS.md`.
- **Two-tab model** (WORKFLOW.md §Two-tab model): **planning tab** = bd issues/epics/dep edges + doc mirrors (`ROADMAP.md`, `plans/epic-*.md`) only — never touches source, commits docs immediately; FULL-lane beads carry frozen contract (`--design`/`--acceptance`). **Dev orchestrator** = all dev, **one bead = one worktree = one branch**; up to **3 independent beads in flight** via background builders — merges stay serial; main checkout is integration-only, source lands via `git merge --ff-only`. History stays linear. Spikes use `spike/<n>` branches — merged or discarded on verdict but the branch is never deleted (permanent record); only `bead/*` branches get `branch -d`. bd works from any worktree (shared DB); `.beads/issues.jsonl` is a passive export — never hand-merge.
- **Commits:** conventional, scoped by bead hash minus the project prefix — `fix(duv): …`, `feat(srs.7): …`; stage explicit paths (never `git add -A`), `--amend` OK on a bead branch but **never on `main`**; pushes need an explicit ask.
- **Delegation** (WORKFLOW.md §Model & agent delegation): main thread plans/joins/reviews; **Sonnet** subagent for a bounded module+test or mechanical edits, **Haiku** for read-only explore. **Agent width = recursion guard:** leaf work → a narrow agent lacking the Agent tool (builder / investigator / reviewer / groomer, defined in `.claude/agents/`); never `general-purpose` under `auto`. Name every spawn (`<bead>-<short>-<model>`), reap on unit close; **idle enforcer** — recall an incomplete subagent with what's missing, never report a step done while a task is open.

## Answering the user

Make the *shape* of every reply legible:

- **Status / done** — plain text stating what finished. No question.
- **Free-text needed** — plain text asking an open-ended question (name something, describe intent, paste a value).
- **Closed choice** — a decision among known, enumerable options: **always** ask via the **AskUserQuestion** tool so options are selectable — never bury a pick-one question in prose.

**Don't over-compress.** Several distinct decisions → several selectable questions (AskUserQuestion batches up to 4) or sequential turns — never force-fit unrelated choices into one oversimplified question.

## Key files
{{Fill as the codebase grows; move the full per-file index to docs/key-files.md and keep a 5-bullet orientation here: state, main orchestration point, persistence, shared utils.}}

## Conventions
{{CONVENTIONS — the enforceable house rules, e.g.:}}
- {{CSS approach / design tokens / theming rule}}
- {{state-management rule}}
- {{component/dir grouping rule}}

## Business rules

**Per-feature implementation deep-dives live in [`docs/features/`](./docs/features/)** — read the relevant one before touching that subsystem.

### Invariants (the gotchas — don't rederive; full context in the linked feature doc)
{{Accrete these as they're discovered — one bullet per invariant, each linking its feature doc. These are the rules that were expensive to learn; keeping them here is what stops re-learning them.}}

## Problems log
Two files in [`problems/`](./problems/): [`PROBLEMS.md`](./problems/PROBLEMS.md) is the one-line TLDR index; [`PROBLEMS_DETAILS.md`](./problems/PROBLEMS_DETAILS.md) holds the full root-cause/fix write-ups. **Whenever a bug is found and resolved, append BOTH — the full entry in PROBLEMS_DETAILS.md and a linked one-liner in PROBLEMS.md, same number.**

**Every bug fix ships with a unit test.** Add a test that fails on the old behavior and passes on the fix — same commit as the fix. If the buggy logic isn't unit-testable as written (buried in a component/effect), extract the decision into a pure function, test that, and wire it back. No fix is "done" without it.

## Roadmap
Feature ideas tracked in [`ROADMAP.md`](./ROADMAP.md). **When implementing a numbered item, mark it checked (`[x]`) and include the change in the commit.**

## Plans
Implementation plans live in [`plans/`](./plans/) — one file per phase/feature. Update the plan file as steps complete; commit plan updates alongside the step they describe.

**Every time a bd epic is created, mirror it in docs:** add a brief `plans/epic-*.md` (why, scope table with child bd ids, per-item checklist, build order) **and/or** update ROADMAP.md. bd holds task state; the `.md` holds the human-readable plan. A bd epic with no plan/roadmap entry is a gap.

## Beads Issue Tracker

This project uses **bd (beads)** for issue tracking. Run `bd prime` for full workflow context.

```bash
bd ready              # Find available work
bd show <id>          # View issue details
bd update <id> --claim  # Claim work
bd close <id>         # Complete work
```

Rules:
- Use `bd` for ALL task tracking — do NOT use TodoWrite, TaskCreate, or markdown TODO lists
- Use `bd remember` for persistent cross-session knowledge
- `.beads/issues.jsonl` is a passive export — never hand-merge; issues live in the local Dolt DB
- Conservative git profile: no pushes or `bd dolt push` without an explicit ask. One-commit-per-build-step is standing commit authority (see WORKFLOW.md).
