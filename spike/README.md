# spike/ — throwaway proof-of-assumption code

Everything in this directory except this README is **gitignored on purpose**.

- A spike proves or kills a risky assumption *before* production work.
- The deliverable is never the code — it is `docs/spike-*.md` with findings
  and a **GO / PARTIAL / NO-GO** verdict (template: `docs/spike-TEMPLATE.md`).
- Spikes run on a `spike/<n>` branch in their own worktree. On verdict:
  merge or discard the useful parts, remove the worktree, `bd close` —
  but **never delete the `spike/*` branch ref** (permanent experiment record;
  `branch -d` is for `bead/*` only).
