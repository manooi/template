# SCALING.md — when the single-orchestrator loop feels slow

Read this when throughput drops or WIP feels capped. Distilled from the source
project's scaling analysis (orchestrator.md, 2026-07) plus a later bottleneck
review. Short version: **find the actual constraint before adding machinery** —
it moved twice in the source project and each fix targeted the measured wall,
not the felt one.

## Cycle anatomy (per bead, serial base loop)

| Stage | Wall time | Parallelizable? |
|---|---|---|
| Worktree setup + dep install | 1–2 min | ❌ cold-start per bead |
| Builder edit + self-gate | **5–20 min (dominates)** | ✅ background, WIP 2–3 |
| Orchestrator review | 2–5 min | ❌ serial, one context |
| Rebase + re-gate + ff-only merge | 2–3 min | ❌ serial **by design** |
| bd ceremony + doc mirror | 1–3 min | ❌ attention tax |

## Bottleneck ranking (after the async pipeline is adopted)

1. **Orchestrator attention.** Not the merges — ff-only is seconds and
   single-writer-to-main is deliberate (two sessions writing one tree is the
   failure the two-tab model exists to prevent). The cost is review + all
   coordination cycling through one growing LLM context. Context growth is
   *why* WIP caps at 3 — past that, review quality measurably drops. The cap
   is a property of one-brain coordination, not a policy choice.
2. **Float supply (upstream, quiet, often the true constraint).** Epics
   decompose into serial chains; independence lives *across* epics. The
   pipeline starves unless 2–3 unrelated epics stay groomed simultaneously —
   and fat work orders come from planning sessions with the user, so
   throughput caps at grooming rate. Also audit what "ready" means: beads
   gated on human eyes (visual QA, console ops, device tests) aren't
   machine-doable float.
3. **The user.** Spike verdicts, visual QA, grooming, trust in merges. Any
   scaling that doesn't scale user-review trust just moves the queue.
4. **Tracker ceremony.** bd create/claim/close + epic doc mirror ≈ 1–3
   min/bead. Small wall-clock share but it spends the scarce resource
   (orchestrator attention). Keep — the dep graph is what makes parallel WIP
   safe — but trim: batch bd ops, mirror epics only.
5. **Cold start.** One dep install per worktree, × WIP.

## Diagnostic before any fix

`bd ready` shape over a normal week:
- **3+ genuinely independent, machine-doable beads** → backlog has float →
  concurrency pays (options A/F below).
- **Usually 1 bead** → backlog is chain-shaped → more agents buy nothing;
  fix is planning-side (decompose epics for parallel seams, groom 2–3 epics).

Also measurable: closed beads carry created/claimed/closed timestamps —
mine the claim→close distribution before trusting the anatomy table.

## Options ladder (adopt in roughly this order)

| # | Option | One-liner | Adopt when |
|---|---|---|---|
| A | **Async pipeline, WIP 2–3** | Background builders, several beads in flight; review/merge as they land; merges stay serial | First — as soon as float exists |
| B | **Fat work orders** | Planning freezes contract into the bead (`--design`/`--acceptance`); spawn becomes near-mechanical | With A |
| — | **Deeper review delegation** | Reviewer-agent pre-pass mandatory; orchestrator reads findings + diff-stat, deep-reads only flagged files | When review is the visible queue |
| — | **Delegate grooming (`groomer` agent)** | Orchestrator spawns the groomer to draft decomposition + contracts; light beads approved by orchestrator, epics by the user | When grooming rate caps throughput — usually early; also the fix for chain-shaped backlogs (groomer cuts for float) |
| F | **Warm builder pool** | 2 persistent named builders with warm worktrees, fed via messages; kills cold start | When installs/cold-starts dominate; revises reap-on-close hygiene — needs an explicit WORKFLOW.md edit |
| C | **Integrator split** | Second session drains ready-to-merge branches (rebase, gates, ff-only, close) | Only when the merge queue visibly backs up |
| E | **Workflow fan-outs** | Deterministic script pipelines builder→verifier over an item list | Burst work (migrations, audits); per-run user opt-in |
| D | **N orchestrators, disjoint surfaces** | One orchestrator per epic with a declared file surface | Last resort — re-risks shared-tree clobbering; partitions leak (features cross directories) |

## Failure modes to watch at higher WIP

- **ROADMAP.md conflict noise** — every bead ticks a checkbox; conflicts are
  trivial (keep both) but frequent.
- **Re-gate tax** — each serial merge moves main; next bead rebases. Rule:
  doc-only delta vs the gated tree ⇒ gates carry over; any src delta ⇒ full
  re-run.
- **Review quality decay** — the WIP cap exists because of it. If findings
  get rubber-stamped, lower WIP before raising it.
