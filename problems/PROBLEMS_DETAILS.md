# Known Problems & Fixes — Full Write-ups

Companion to [PROBLEMS.md](./PROBLEMS.md) (the one-line index). Same numbers.
Entry template:

---

## N. {{Symptom title, as the user saw it}}

**Symptom:** {{what was observed — exact error text if any}}

**Root cause:** {{the mechanism, not just the location. Why did it happen?}}

**Fix:** {{what changed, `file:line` refs}}

**Regression test:** {{test name + file. If the logic had to be extracted into
a pure function to be testable, say so.}}

**Alternatives considered:** {{optional — rejected approaches and why}}

---
