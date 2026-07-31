# Epic: {{Title}} (`{{bead-id}}`, ROADMAP #{{N}})

Doc mirror of the bd epic — bd holds task state; this file holds the
human-readable plan. Created in the same session as `bd create` for the epic.

## Why

{{2–4 numbered findings/motivations. Cite evidence: file:line, measurements,
user reports, research dates. This section is what makes the epic
re-evaluatable in six months.}}

## Decision (pending spike, if any)

{{The chosen approach in 2–5 bullets: format/library/API picks and the one-line
reason each. If a spike gates this, say "pending spike `{{id}}.1`".}}

## Scope

| bd id | What | Depends on |
|---|---|---|
| `{{id}}.1` | Spike: {{risky assumption}} — `docs/spike-*.md`, GO/PARTIAL/NO-GO | — |
| `{{id}}.2` | {{pure module + tests}} | {{id}}.1 |
| `{{id}}.3` | {{second module, parallelizable}} | {{id}}.1 |
| `{{id}}.4` | Wire {{integration}} | {{id}}.2, {{id}}.3 |
| `{{id}}.5` | (P3) {{stretch}} | {{id}}.4 |

## Checklist

- [ ] `{{id}}.1` Spike — verdict written
- [ ] `{{id}}.2` …
- [ ] `{{id}}.3` …
- [ ] `{{id}}.4` …
- [ ] `{{id}}.5` …

## Build order

`{{id}}.1` (spike gate) → `{{id}}.2` ∥ `{{id}}.3` (parallelizable —
contract-first: freeze the shared signatures before spawning) → `{{id}}.4`
(integration, main thread) → `{{id}}.5` (post-core).

## Invariants to respect

{{Existing business rules this epic must not break — pull from CLAUDE.md
§Invariants and the relevant docs/features/*.md. Listing them here is what
keeps a builder from re-learning them the hard way.}}
