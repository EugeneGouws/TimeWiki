---
tags: [concept, timemath, explored]
created: 2026-07-26
updated: 2026-07-26
sources: [raw/sources/timemath/solver-docs/docs/explorations/phase4-teacher-min.md, raw/sources/timemath/solver-docs/PLAN.md]
---

# Phase 4 — teacher assignment, cost of minimising distinct teachers

**Status: EXPLORED, NOT IMPLEMENTED.** Question: minimal teacher usage is
the product goal (see [[timemath-project]]) — what does *proving* minimality
cost, and is it worth paying?

## Findings

* Proving optimality (fewest distinct teachers) took up to 38s even on small
  instances — feasibility is cheap, the optimality proof is not, and it gets
  worse at scale.
* Motivated a product-goal reframe: find a feasible assignment with
  available teachers, **or** compute a minimum-cost repair when infeasible —
  skip the optimality proof in the common feasible case. See
  [[repair-model]].
* Phase 3↔4 coupling is a real, confirmed risk: a section-cheap column
  assignment (Phase 3) can force more distinct teachers (Phase 4).
  Unresolved: fold teacher variables into Phase 3, or iterate 3↔4 with
  feedback.
