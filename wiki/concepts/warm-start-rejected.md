---
tags: [concept, timemath, explored, rejected]
created: 2026-07-26
updated: 2026-07-26
sources: [raw/sources/timemath/solver-docs/docs/explorations/warm-start.md]
---

# Warm-starting from a prior year's timetable — rejected

**Status: EXPLORED, REJECTED.** Question: can last year's timetable seed
this year's solve, for [[timemath-project]]?

**No.** Year-to-year cell-placement Jaccard similarity is only ~0.19 despite
stable structure (columns, bands, M-values) — the solution surface moves
almost completely even when the problem barely does. Worse, a warm start
biases the solver toward defending last year's teacher count, which is
exactly the number Phase 4 is trying to minimise (see
[[phase4-teacher-assignment]]).

If hints are wanted, they must come from Phase 2's own heuristic placements
(`AddHint`), never from a prior year's solution.
