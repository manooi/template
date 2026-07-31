# Spike #{{N}} — {{risky assumption being tested}}

**Verdict: {{GO / PARTIAL / NO-GO}}.** {{One paragraph: what was proven or
killed, and what happens next. GO → proceed to the build step, with any
format/API corrections below. PARTIAL → proceed with named limitations.
NO-GO → epic re-planned or dropped; say why.}}

Tested against {{concrete artifact: real file, real API, real device}} using
{{exact library versions}}.

---

## Findings

### 1. {{Finding title}}

{{Evidence-first: numbers, op counts, actual API shapes, worked examples.
Where the spike contradicts the plan's assumption, say so explicitly —
"the plan assumed X; actual shape is Y" — those corrections are the spike's
main payload.}}

### 2. {{Finding title}}

…

## Acceptance checks

| Check | Result |
|---|---|
| {{check from the spike bead's acceptance criteria}} | ✅ / ❌ {{detail}} |

## Rules for the build step

{{Bulleted list of corrections/constraints the production implementation must
follow, distilled from the findings. These usually graduate into
CLAUDE.md §Invariants once the feature ships.}}

---

*Spike branch: `spike/{{n}}` — permanent record, never deleted. Throwaway code
lives under gitignored `spike/`; merge or discard on the verdict.*
