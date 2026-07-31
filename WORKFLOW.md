# WORKFLOW.md — how we build in this repo

The process rules, made explicit. Read this before starting a multi-step feature. CLAUDE.md links here.

Throughout: `<gates>` = this repo's quality gates ({{GATE_CMDS, e.g. `pnpm run build && pnpm test` + lint}}); `<repo>` = this repo's directory name.

## Feature lifecycle

1. **Roadmap item** — every feature is a numbered `#N` in `ROADMAP.md`.
2. **Plan** — non-trivial features get a `plans/*.md` file: steps, estimates, verify strategy, risks. Update the plan as steps land.
3. **Spike-gate the risky assumption** — if a feature rests on an unproven assumption (a library exposes the data we need, a format round-trips, coordinates map correctly), a throwaway spike proves or kills it *before* production work. Spike deliverable = `docs/spike-*.md` with a findings table and a **GO / PARTIAL / NO-GO** verdict. Do not start the build until the verdict is GO/PARTIAL.
4. **Build, one commit per step** — each production step is one commit that leaves the app working and ticks the ROADMAP checkbox in the same commit. Update the plan file alongside. This is standing commit authority for build steps; pushes and `bd dolt push` still require an explicit ask. Commit subject = conventional-commit scoped by the bead hash without the project prefix (`feat(srs.7): show group label beside each marker`).
5. **Log problems** — bugs found and fixed go in `problems/PROBLEMS.md` + `problems/PROBLEMS_DETAILS.md`, always.

## Two-tab model — planning tab + dev orchestrator (worktree-per-bead)

At most two sessions touch this repo at once, with **disjoint write surfaces** (shared-tree clobbering between concurrent sessions is the failure this prevents):

- **Planning tab** (user + Claude): bd work only — create issues/epics/dep edges — plus their doc mirrors (ROADMAP items, `plans/epic-*.md`). **Never touches source. Never merges.** Commits its doc edits immediately — a dirty ROADMAP.md in the main checkout blocks the orchestrator's next merge. **Fat work orders:** FULL-lane beads carry their frozen contract at creation — `--design` (types, signatures, file boundaries) and `--acceptance` (verify criteria) — so the orchestrator's spawn is near-mechanical. Light beads may be self-groomed by the orchestrator.
- **Dev orchestrator tab** (one session): all development. Decomposes beads (contract-first, §Parallel work), spawns worktree-isolated subagents, reviews, owns **every** join to `main`, closes beads. The main checkout is its integration station — source changes land there **only via merges**.

**All dev work happens on a bead branch in its own worktree.** One bead = one worktree = one branch.

### Layout

- Worktrees live in a sibling directory: `../<repo>-wt/<bead-id>` (outside the repo — no `.gitignore` entry needed, no dev-server watcher noise).
- Branch name = `bead/<bead-id>` (child issues keep the dot: `bead/xyz.7`).
- Main checkout (always on `main`): merges, `git worktree add/remove`, bd ops, planning-tab doc commits. Nothing else.

### Orchestrator loop

```bash
# 1. START
bd ready                                   # pick a bead
bd update <id> --claim
git worktree add ../<repo>-wt/<id> -b bead/<id> main     # RUN FROM MAIN CHECKOUT (path is cwd-relative)
# decomposable feature → freeze contract, spawn subagents (§Model & agent delegation)
#   with isolation: "worktree"
cd ../<repo>-wt/<id>/... && <install deps>               # once per worktree

# 2. BUILD — on the bead branch, inside its worktree
#    One-commit-per-step; tick the ROADMAP checkbox in the same commit.
#    --amend allowed here (branch is private) — never on main.

# 3. JOIN — orchestrator only, one bead at a time
git rebase main                            # in the worktree; conflicts resolve here, not on main
<gates>                                    # must be green AFTER the rebase
#    then, FROM THE MAIN CHECKOUT (running it inside the worktree = silent no-op):
git merge --ff-only bead/<id>
#    ff-only fails ⇒ main moved (planner doc commit) ⇒ rebase again.

# 4. CLEAN UP
git worktree remove ../<repo>-wt/<id>
git branch -d bead/<id>
bd close <id>
```

### Async pipeline — WIP 2–3 across beads

The loop above is the base case. When `bd ready` holds 2–3 **independent** beads — no dep edges, disjoint file surfaces — the orchestrator may run them concurrently:

1. **Claim + spawn in a burst.** For each bead: claim, `git worktree add`, spawn its builder with `run_in_background: true` (named `<bead>-<short>-<model>`). **WIP cap 3** — above that, review quality drops.
2. **React to completions.** The harness notifies as each builder returns. Review that bead's diff (reviewer-agent pre-pass for bigger diffs), then JOIN it while the other builders keep running.
3. **Merges stay strictly serial** — one bead at a time through the main checkout. Each merge moves `main`, so the next bead re-joins via rebase. **Re-gate rule:** after a rebase, if the delta vs the tree you already gated is doc-only (`git diff <gated-sha> HEAD --stat -- <src paths>` is empty), gates carry over; any src/tooling delta ⇒ full re-run.
4. **Independence is the gate.** Shared files or a dep edge ⇒ serial, not parallel. When unsure, serial.

Reap-on-completion: every background builder is shut down when its bead closes.

### Rules & facts

- **bd works from inside any worktree** — it resolves the main repo's DB. `.beads/issues.jsonl` stays a passive export — never hand-merge it.
- **ROADMAP.md is the one shared file**: planner adds items (on `main`), dev ticks checkboxes (on bead branches). Occasional rebase conflict there is expected and trivial — keep both edits.
- **Epics**: children worked serially share one worktree/branch (`bead/<epic-id>`); children worked in parallel get a worktree each; dep edges in bd (`bd dep add`) define the join order.
- **Never on `main`**: `--amend`, force-push, `git add -A`/`git add .`. On a bead branch inside its own worktree, `--amend` is safe; explicit-path staging stays the habit everywhere. `git reflog` is the recovery net.
- **Each worktree needs its own dependency install.**
- **Spikes**: `spike/<n>` lives in its own worktree too; contents under gitignored `spike/` are throwaway — merge or discard on the verdict, but **the branch ref itself is never deleted** (permanent record); cleanup is worktree remove + `bd close` only, no `branch -d`.
- **Rebase-then-ff-only keeps history linear** — trunk-based, short-lived branches, no merge commits.

## Model & agent delegation (selective)

Standing policy — no need to instruct per-task. Main thread decides, and delegates **only when it pays**.

- **Main thread (strongest model)** — planning, spike verdicts, critical-path decisions, integration/wiring across files, code review, anything needing whole-repo context.
- **Sonnet subagent (`builder`, `.claude/agents/builder.md`)** — a bounded, well-specified implementation task: one module + its test, against a **frozen contract** (types + signatures agreed first). Mechanical edits (renames, format tweaks) also go here.
- **Haiku subagent (`investigator`, `.claude/agents/investigator.md`)** — **read-only exploration only**: locating code, grep-maps, codebase surveys. Never write/edit/commit tasks.
- **Reviewer (`reviewer`, `.claude/agents/reviewer.md`)** — diff pre-pass on bead branches: scope check first, then correctness; one line per finding.
- **Groomer (`groomer`, `.claude/agents/groomer.md`)** — the BA role: decided intent → decomposed beads with dep edges + frozen `--design`/`--acceptance` contracts + epic doc mirror. Decides HOW, never WHAT — ambiguity returns as questions. **Approval gate:** groomed contracts are drafts until approved — the orchestrator may approve light beads; epic contracts need the user. Cuts for float (independent children) so the async pipeline stays fed.

**Delegate only when** the task is (a) bounded and independent, OR (b) parallelizable with other work, OR (c) large enough that the subagent's cold-start cost is less than the cheaper-model saving. **Do not delegate** tiny 1-line edits or anything needing context the main thread already holds. When unsure, main thread reviews inline but the *edit* still goes to a builder (see hands-off rule).

### Tier escalation — same agents, stronger model when the task warrants

`builder` and `reviewer` default to **sonnet** (pinned in their frontmatter). For hard tasks the orchestrator escalates by passing a spawn-time `model` override — the agent definition (tools, rules, output contract) stays identical; only the model changes. One file per role, no `-hard` variants: escalation is a deliberate per-spawn decision, never a different prompt.

| Escalate to **opus builder** when | Escalate to **opus reviewer** when |
|---|---|
| Subtle algorithm / tricky math or geometry | Diff is large or touches many files |
| Concurrency, async races, ordering bugs | Core invariants / shared contracts on the surface |
| Cross-cutting change threading many call sites | Security- or data-loss-sensitive change |
| Security- or correctness-critical logic | Pre-release / pre-deploy audit pass |
| **A sonnet builder missed the contract twice** — re-spawn on opus with the same contract + a note on both failures, don't keep recalling | Sonnet review and orchestrator read disagree — tie-break |

Rules: the spawn name always carries the **actual** model (`<bead>-<short>-opus`); state the tier when reporting the spawn. Default stays sonnet — escalation needs a reason you could say out loud. Haiku never escalates (exploration stays cheap; if exploration needs judgment, that's the main thread's job). `groomer` escalates to opus for multi-epic decomposition or architecture-heavy contracts — contract errors amplify downstream, so grooming earns escalation sooner than building does.

### Agent hygiene — spawn light, tag the model, reap when done

- **Default spawn = one-shot foreground subagent** — no `name`, no `run_in_background`, not a teammate. It returns its result and terminates. Later recall still works via `SendMessage` to the spawn's `agentId`. **Sanctioned exception — async pipeline:** builders for independent beads run `run_in_background: true`, named, WIP ≤ 3, reaped on bead close.
- **Persistent forms need a reason.** Background only when the main thread must keep working in parallel; named teammates only for genuinely interactive coordination. Whoever spawns a persistent agent owns shutting it down.
- **Reap on completion.** Once a delegated unit is verified and `bd close`d, stop its background task. End-of-feature sweep: zero idle agents — same gate as "zero open bd issues".
- **Model visibility — no anonymous workers.** Every spawned agent's name carries its model: `<bead>-<short>-sonnet`. State the model when reporting a spawn and its result. **Always pass `model` explicitly** — agents inherit the parent's (expensive) model when omitted.
- **Agent width = task shape — recursion guard.** `general-purpose` carries `Tools: *`, which includes the Agent tool; under `auto` it will decompose a leaf task and spawn its own unnamed sub-builders, which orphan (no name → no shutdown handle). Fix is structural, not a prompt plea: for leaf/bounded work spawn an agent whose tool grant LACKS Agent — this repo defines `builder` / `investigator` / `reviewer` in `.claude/agents/` exactly for this (models pinned in frontmatter, gates baked into the builder prompt); built-in `Explore` also qualifies. Reserve `general-purpose` for genuinely unknown-shape work; never run a fan-out-capable agent under `auto`. **Cap capability with tool grants, not instructions.**

### Orchestrator hands-off rule — never edits files itself

The dev-orchestrator main thread **never edits files** — not source, not docs, not throwaway debug instrumentation. Every file change is a subagent deliverable:

- **Builder** (Sonnet) for code and docs.
- **Investigator** for instrument-run-revert debugging — the agent adds temp logging, runs, reports, and reverts.
- **Reviewer/verify agent** for verification runs.

The orchestrator's own tools stay **read-only** (read/grep/diff review) plus **git integration** (worktree/rebase/merge/branch), **bd**, and **agent lifecycle**. No exceptions for "just a one-liner."

## Parallel work & the critical path (construction-PM model)

Break a multi-file feature into a dependency graph, borrowing critical-path method:

- **Critical path** = the longest chain of dependent tasks. Sets the minimum time. Main thread owns it.
- **Float** = tasks with no dependency between them. The parallel candidates — one subagent each.
- **Integration points** = where parallel work joins (the "trade clash" risk). Minimise them; the main thread does the join.

**Contract-first rule (the shop-drawing discipline):** before any parallel agent starts, freeze the shared surface — types, function signatures, file boundaries. Agents build against the contract, so they don't clash at the join. This is the single thing that makes parallelism safe.

**Isolation:** parallel agents that touch the working tree run with `isolation: "worktree"`.

**When to actually parallelize:** only when float tasks are genuinely independent AND sizable (a full module + tests each). Steps touching 1–2 files — solo is faster. Don't parallelize for its own sake.

```
[freeze contract: types + signatures]      ← critical-path head — main thread
        │
   ┌────┴────┐                              ← float — parallel subagents (worktree-isolated)
   ▼         ▼
moduleA      moduleB      (each: impl + test, one Sonnet agent)
   └────┬────┘
        ▼
[wiring / integration]                      ← critical-path tail — main thread (owns the join)
        ▼
[E2E verify]
```

## Task tracking & idle enforcer

Task tracking lives in **bd (beads)** — never TodoWrite/TaskCreate/markdown TODO lists.

- **One bd issue per delegated unit.** `bd create` each (with `--parent=<epic-id>`); encode the dependency graph with `bd dep add <task> <blocker>` — those edges ARE the critical path (`bd ready` lists only unblocked issues).
- **Verify before close.** A subagent reports its unit done, but the main thread confirms the contract was actually met (tests pass, file exists, signatures match) before `bd close`. Never close on a partial or failing result.
- **Idle enforcer — recall, don't drop.** If a subagent returns without its task completed (partial, tests red, wandered off-scope), the issue stays `in_progress`. Recall the same agent via `SendMessage` stating exactly what's missing, rather than re-spawning cold or quietly finishing it yourself. Re-delegate only if truly wedged.
- **Feature-done gate.** Do not report a step/feature complete while any of its bd issues is open. `bd list --status=open` scoped to the epic is the checklist.

## Doc hygiene

- CLAUDE.md is loaded every session — keep additions terse; push detail into plan/spike/feature docs and this file.
- Feature invariants that were expensive to learn get one bullet in CLAUDE.md §Invariants + a full explanation in `docs/features/<area>.md`.
