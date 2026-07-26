---
tags: [concept, timemath, combinatorics]
created: 2026-07-26
updated: 2026-07-26
sources: [raw/sources/timemath/CLAUDE_root.md, raw/sources/timemath/solver-docs/CLAUDE.md, raw/sources/timemath/solver-docs/docs/explorations/phase3-feasibility.md]
---

# Basket-level SDR (System of Distinct Representatives)

The binding constraint in [[timemath-project]], established as more
fundamental than pairwise subject conflict. Two subjects clash only if some
*actual student basket* contains both. A basket of *k* choice subjects needs
*k* distinct columns; by Hall's theorem this is satisfiable iff every
sub-collection of *j* subjects has ≥ *j* columns available between them.

## Soundness boundary (load-bearing correction)

Subject-level all-different is sound **only for floor=1 (singleton-section)
subjects** — a genuine pairwise all-different. For subjects with floor ≥ 2,
students can legally split across parallel sections in different columns;
forcing subject-level distinctness on them **removes real solutions**. Those
subjects contribute to the Hall *counting* check only, never to pairwise
elimination.

This boundary broke the original Phase 3 model (PLAN.md §5, subject-level
SDR) once any subject ran multiple sections — see
[[phase3-column-student-assignment]] for the corrected `y[student,section]`
formulation that replaced it.

## Where it's implemented

* Phase 2 (precolour): propagate basket-SDR + teacher-exclusivity to
  fixpoint, split by the floor=1/floor≥2 boundary above. Domain collapse to
  empty fails with a named Hall violator.
* Phase 3: per-student all-different on assigned sections' columns is the
  hard constraint that *replaced* basket-level subject SDR.
