# Project Scaffold — the process engine

This template captures a working process battle-tested in a real production
project (483 bd issues, 434 closed, ~80 logged root-caused bugs, linear git
history throughout). It is the *process engine*, stripped of anything
project-specific.

## What's in here

Grouped by role — everything above the fold is at repo root; the rest is in
its named directory.

| File | What it is |
|---|---|
| **Entrypoints** *(root, read first)* | |
| `README.md` | This file — what the scaffold is and how to instantiate it |
| `INSTALLATION.md` | Agent-facing init runbook — hand the repo to an AI agent and it self-initializes (interview → fill placeholders → wire gate → init bd) |
| `CLAUDE.md` | Project-instructions template — fill the `{{PLACEHOLDERS}}`, delete what doesn't apply |
| `AGENTS.md` | Cross-tool pointer at CLAUDE.md (symlink or mirror) |
| **Playbook** *(root, read once, keep forever)* | |
| `WORKFLOW.md` | The full playbook: lanes, two-tab model, worktree-per-bead, delegation, parallelism |
| `SCALING.md` | What to do when the serial loop feels slow — bottleneck ranking, diagnostic, options ladder |
| `LESSONS.md` | Hard-won gotchas from the source repo — read once, keep forever |
| **Living state** *(root, both tabs write it)* | |
| `ROADMAP.md` | Numbered-item feature backlog skeleton |
| **Templates & logs** *(in named dirs)* | |
| `problems/PROBLEMS.md` / `problems/PROBLEMS_DETAILS.md` | Dual-file bug log skeleton (TLDR index + full write-ups) |
| `plans/epic-TEMPLATE.md` | Epic doc-mirror template (why / scope table / checklist / build order) |
| `docs/spike-TEMPLATE.md` | Spike report template (findings → GO/PARTIAL/NO-GO verdict) |
| `docs/features/README.md` | Per-feature deep-dive convention |
| `docs/key-files.md` | Per-file index stub |
| **Tooling** | |
| `.claude/settings.json` | SessionStart `bd prime` hook + read-only git/bd permission allowlist |
| `.gitignore` + `spike/README.md` | Spike-code-is-throwaway convention, pre-wired |
| `.claude/agents/` | `builder` / `investigator` / `reviewer` / `groomer` — tool-narrowed subagents (no Agent tool = structural recursion guard), sonnet pinned as default with per-spawn opus escalation (WORKFLOW.md matrix), gate commands templated into the builder prompt; groomer = BA role drafting bead contracts behind an approval gate |

## The core ideas (why this works)

1. **No bead, no code.** Every change starts as a tracked issue with a claim.
   Nothing lands untracked; `bd ready` is always the honest work queue.
2. **Orchestrator never edits.** The main session plans, spawns, reviews,
   merges. All file edits are subagent deliverables. Keeps the orchestrator's
   context cheap and its judgment unbiased by its own code.
3. **One bead = one worktree = one branch.** Rebase → gates → `--ff-only`
   merge from the main checkout. History stays linear; main is never broken.
4. **Spike-gate risky assumptions.** Throwaway spike with a written
   GO/PARTIAL/NO-GO verdict *before* production work. Kills bad bets cheap.
5. **Contract-first parallelism.** Freeze types/signatures/file boundaries
   before spawning parallel builders (construction-PM: critical path vs float).
6. **Every bugfix ships a regression test, same commit.** If the logic isn't
   testable, extract a pure function and test that.
7. **Docs mirror the tracker.** bd holds state; markdown holds the
   human-readable plan (epic files, roadmap checkboxes ticked in the same
   commit as the change). A bd epic with no plan file is a gap.
8. **Log every solved bug** (dual-file: one-liner index + full root-cause).
   The index becomes the project's immune system.

## Instantiating a new project

**Fastest path — let an AI agent do it:** point a coding agent at this repo and
tell it to read [`INSTALLATION.md`](./INSTALLATION.md). It interviews you,
resolves every placeholder, wires the pre-commit gate, inits bd, and leaves a
clean first commit. The manual steps below are the same work by hand.

```bash
# 1. Copy the scaffold into the new repo root
cp -R ~/template/{CLAUDE.md,AGENTS.md,WORKFLOW.md,SCALING.md,LESSONS.md,ROADMAP.md,problems,plans,docs,spike,.claude,.gitignore} <new-repo>/

# 2. Fill placeholders
#    CLAUDE.md: {{PROJECT_ONELINER}}, {{REPO_LAYOUT}}, {{DEV_CMDS}}, {{GATE_CMDS}}, {{CONVENTIONS}}
#    WORKFLOW.md: replace <repo> and <gate commands>
#    .claude/agents/builder.md: {{GATE_CMDS}} — must match the pre-commit hook exactly (gate parity)

# 3. Init beads
cd <new-repo> && bd init            # then: bd create your first epic

# 4. Pre-commit gate (husky or equivalent) — lint + typecheck, e.g.:
#    pnpm exec lint-staged && pnpm exec tsc -b
#    (gate must match what builders run — see LESSONS.md "gate parity")

# 5. Worktree parent dir is created on demand by `git worktree add ../<repo>-wt/<id>`
#    — just keep the sibling-name convention.

# 6. If you use AGENTS.md for other tools: symlink it to CLAUDE.md or
#    mirror substantive edits both ways.
```

The scaffold is self-contained: the three subagents (`builder` /
`investigator` / `reviewer`) ship in `.claude/agents/` and depend on no
plugin. Their prompts already enforce compressed output (receipts, findings
tables, one-line-per-finding) — the context-economy benefit travels with the
repo.
