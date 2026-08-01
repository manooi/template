# INSTALL — instructions for an AI agent

**You are an AI coding agent. A human handed you this repo and asked you to
initialize a new project from it. This file is your runbook. Follow it top to
bottom.**

This repo is a **process scaffold** — a battle-tested working process (bead
tracking, worktree-per-task, subagent delegation, spike gates, dual-file bug
log) with all project-specific content replaced by `{{PLACEHOLDER}}` tokens.
"Initialize" means: resolve every placeholder to a real value for *this*
human's project, wire the quality gate, start the issue tracker, and leave a
clean first commit.

Do **not** start writing feature code. Your job is setup only.

---

## Step 0 — Detect the mode

Two ways this repo reaches the human. Figure out which, then act.

| Mode | Signal | What to do |
|---|---|---|
| **In-place** | This scaffold *is* the project root; the human will build here | Fill placeholders directly in these files |
| **Copy-out** | The scaffold is a reference; the real project lives elsewhere | Copy the scaffold files into the target repo root first (command below), then fill there |

Copy-out command (run from the scaffold, adjust `<new-repo>`):

```bash
cp -R ./{CLAUDE.md,AGENTS.md,WORKFLOW.md,SCALING.md,LESSONS.md,ROADMAP.md,problems,plans,docs,spike,.claude,.gitignore} <new-repo>/
```

Ask the human which mode if it is not obvious from context.

---

## Step 1 — Interview the human

You cannot invent these answers — they encode product and stack decisions.
Ask them **before** editing anything. Batch the questions; keep it short.

1. **Project one-liner** — what the app is, the stack, who uses it.
2. **Repo layout** — top-level dirs, and *where dev commands run* (e.g.
   `frontend/app/`). One bullet per dir.
3. **Dev commands** — how to run / build / test (e.g.
   `pnpm run dev | build | test`).
4. **Gate commands** — the exact lint + typecheck + test commands the
   pre-commit hook will enforce. ⚠️ These must match what builders self-run.
   See the gate-parity warning in Step 4.
5. **Conventions** — the enforceable house rules (CSS/theming approach, state
   management, component grouping — whatever applies).
6. **Package manager / runtime** — pnpm, npm, bun, cargo, go, etc. (drives the
   gate and hook wiring).
7. **First epic / roadmap item** — one real piece of work to seed the tracker.

If an answer does not apply to their stack, that placeholder gets **deleted**,
not filled with filler.

---

## Step 2 — Resolve every placeholder

Search the repo for unresolved tokens and replace each with a real value from
the interview. Nothing may ship with a `{{...}}` still in it (except inside the
`*-TEMPLATE.md` files and `spike-TEMPLATE.md` / `epic-TEMPLATE.md`, which stay
as templates on purpose).

```bash
grep -rn '{{[^}]*}}' --include='*.md' --include='*.json' . | grep -v '.git/'
```

Placeholder map — where each lives and what it becomes:

| File | Token(s) | Replace with |
|---|---|---|
| `CLAUDE.md` | `PROJECT_ONELINER` | Interview #1 |
| `CLAUDE.md` | `REPO_LAYOUT`, `DEV_CMDS`, the `frontend/app` / `backend` example bullets | Interview #2, #3 — real dirs, real commands |
| `CLAUDE.md` | `{{REPO}}` (worktree path example) | The repo's directory name |
| `CLAUDE.md` | `CONVENTIONS` + the three convention sub-bullets | Interview #5 (delete unused) |
| `CLAUDE.md` | Key-files stub, Invariants stub | Leave as guidance; fill as code grows |
| `WORKFLOW.md` | `GATE_CMDS`, `<repo>`, `<gates>` | Interview #4, repo name |
| `.claude/agents/builder.md` | `GATE_CMDS` (line ~28) | Interview #4 — **must be identical** to the pre-commit hook (gate parity) |
| `ROADMAP.md` | `#1`..`#5` items, `Area A/B` | Interview #7 + real areas; delete the rest |
| `AGENTS.md` | — | Either `ln -sf CLAUDE.md AGENTS.md` or keep the mirror stub |

Leave these as-is — they are reusable templates, not placeholders to fill now:
`plans/epic-TEMPLATE.md`, `docs/spike-TEMPLATE.md`, `docs/key-files.md`,
`problems/PROBLEMS*.md` skeletons.

---

## Step 3 — Initialize the issue tracker (beads)

The whole process runs on `bd`. No tracker, no lane.

```bash
bd init            # creates the local tracker DB
bd prime           # loads full workflow context — read it
```

Then seed the first real work item from Interview #7:

```bash
bd create "<first epic or roadmap item>" --type epic   # or --type task
```

If it is an epic, mirror it in docs: copy `plans/epic-TEMPLATE.md` to
`plans/epic-<slug>.md`, fill it, and tick the matching `ROADMAP.md` item.
A bd epic with no plan file is a gap.

`.beads/issues.jsonl` is a passive export — never hand-edit or hand-merge it.

---

## Step 4 — Wire the pre-commit gate

Install a pre-commit hook (husky or equivalent) that runs the Interview #4
gate commands, e.g.:

```bash
# example — adapt to the human's stack
pnpm exec lint-staged && pnpm exec tsc -b && pnpm test
```

> ### ⚠️ Gate parity — the one rule that bites
> The commands in the **pre-commit hook** and the `GATE_CMDS` in
> **`.claude/agents/builder.md`** must be **byte-for-byte the same set**.
> Builders self-gate before reporting done; if the hook runs a check the
> builder did not (a lint the builder skipped, say), work reported "green"
> dies at commit time. Fix both places together, always.

---

## Step 5 — Verify, then commit

Self-check before you call this done:

- [ ] `grep -rn '{{[^}]*}}' --include='*.md' --include='*.json' .` returns
      **only** the intentional `*-TEMPLATE.md` template tokens.
- [ ] `bd ready` runs and shows the seeded work.
- [ ] The pre-commit hook exists and its commands equal builder.md `GATE_CMDS`.
- [ ] `CLAUDE.md` reads as a real project doc — no scaffold language left.
- [ ] `AGENTS.md` points at (or symlinks) `CLAUDE.md`.

Then commit the initialized scaffold as one clean commit (conventional
message, stage explicit paths — never `git add -A`):

```bash
git add CLAUDE.md AGENTS.md WORKFLOW.md ROADMAP.md .claude/ plans/ docs/ <others>
git commit -m "chore: initialize project from scaffold"
```

Do **not** push unless the human explicitly asks.

---

## Step 6 — Hand back

Report to the human as a short receipt, not prose:

- Placeholders resolved (count) / any left deliberately unfilled.
- Tracker: `bd init` done, first item = `<id>`.
- Gate: hook installed, commands = `<...>`.
- Next action they should take (usually: `bd ready`, then start the first
  bead per `WORKFLOW.md`).

After this, stop. Building the first feature is a separate, bead-tracked task —
read `WORKFLOW.md` and pick the lane before any `src/` change.

---

*Deep context lives in `README.md` (why this process works), `WORKFLOW.md`
(the full playbook), and `LESSONS.md` (hard-won gotchas). Read them if any step
above is ambiguous.*
