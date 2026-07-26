---
tags: [concept, timemath, explored]
created: 2026-07-26
updated: 2026-07-26
sources: [raw/sources/timemath/solver-docs/docs/explorations/phase3-feasibility.md, raw/sources/timemath/solver-docs/PLAN.md]
---

# Phase 3 — column + student assignment

**Status: EXPLORED, NOT IMPLEMENTED.** Analytical/throwaway-script check of
whether the Phase 3 model in [[timetable-solver-pipeline]] is sound and fast
enough.

## The correction

The original plan did basket SDR over *subjects* — structurally infeasible
once any subject runs multiple sections (see [[basket-sdr]] soundness
boundary). Replaced with: `y[student, section] ∈ {0,1}`, one section per
chosen subject per student, `Σ students ≤ CAP` per section, and
**per-student all-different** over the columns of that student's assigned
sections. This is the hard constraint that supersedes basket-level subject
SDR.

## Findings

* Speed is a non-issue: feasibility solves in < 1.2s for every grade-year
  combination tested — Phase 3 is not the bottleneck.
* A bespoke forward-checking domain tracker in Phase 2 would re-implement
  CP-SAT's own propagation — use assumption literals in Phase 3 instead of
  building that.
* Caveat: runs used provisional teacher data and hand-prepared inputs —
  timings validate direction only, re-measure once qualification files
  ([[timemath-project]] `TT<year>.xlsx`) are rebuilt.
