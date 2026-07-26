Snapshot of `E:\timemath\CLAUDE.md`, taken 2026-07-26 (second snapshot,
supersedes `CLAUDE_root.md` — that file was captured earlier the same day
by a concurrent ingest session, before this update landed). Per wiki
convention this is a new source, not an edit to the old one.

Full file re-read in full; see `E:\timemath\CLAUDE.md` for current content.
Notable deltas vs. the first snapshot (`CLAUDE_root.md`):

1. New "Relationship to other projects" section — TimeEduSuite firewall,
   `ClashDetector` as the only import. (Matches what the solver-docs staging
   draft already said; now adopted into the authoritative root file.)
2. Corrections list grew from 4 to 8 items. New: (5) five Gr10 rows in the
   2026 workbook share a duplicate student ID; (6) GOVENDER covers 3 people,
   WYLIE covers 2, in teacher data — rebuilt `TT<year>.xlsx` splits them
   (GOVENDER_K/L, WYLIE_C/Z); (7) the 2026 teacher table in the original JSON
   export is a stale copy of 2025's, never source teacher identity from it;
   (8) warm-starting the solver from a prior year is rejected — Jaccard
   similarity ~0.19 despite stable structure, and it biases toward defending
   last year's teacher count, which Phase 4 is trying to minimise.
3. "Current state" now explicitly states: "Phases 3-5 have been explored
   analytically ... Nothing beyond Phase 2 exists as code."
4. Next-steps item 1, "Consolidate solver-docs/ into docs/", is marked
   **struck through and "done 2026-07-26"**: claims `docs/explorations/` and
   `docs/archive/BRAINSTORM.md` were added, this file and `docs/PLAN.md`
   merged with the staging draft, and `solver-docs/` deleted.

**Contradiction found on verification (this ingest, 2026-07-26):** the
filesystem does not match claim 4. `E:\timemath\solver-docs\` still exists
in full (CLAUDE.md, PLAN.md, CONSOLIDATION.md, docs/). `E:\timemath\docs\`
has no `explorations/` or `archive/` subfolder. `docs/PLAN.md` is
byte-identical to the pre-update snapshot already ingested. So: the prose
in CLAUDE.md says the merge happened; the actual files say it hasn't. See
[[solver-docs-consolidation]] for how this is tracked — not resolved here,
flagged for Eugene/Claude Code to reconcile in the timemath repo itself.
