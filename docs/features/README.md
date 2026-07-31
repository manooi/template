# docs/features/ — per-feature implementation deep-dives

One file per shipped subsystem (`pdf-pipeline.md`, `export-import.md`, …).
Read the relevant one **before touching that subsystem**; CLAUDE.md
§Business rules links each.

Each file answers, for its feature:

1. **How it works** — the architecture in a page, with `file:line` anchors.
2. **The invariants** — full context behind each one-line gotcha bullet in
   CLAUDE.md §Invariants (what breaks if violated, which PROBLEMS entry
   discovered it).
3. **Extension points** — where the next change to this area most likely goes.

Lifecycle: created when a feature ships (distilled from its plan + spike docs),
updated when the invariants change. CLAUDE.md keeps only the one-line bullet;
the *why* lives here — that split keeps CLAUDE.md loadable every session.
