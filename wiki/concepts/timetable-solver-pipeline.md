---
tags: [concept, timemath]
created: 2026-07-26
updated: 2026-07-26
sources: [raw/sources/timemath/docs/PRODUCT.md, raw/sources/timemath/docs/PLAN.md, raw/sources/timemath/solver-docs/PLAN.md]
---

# Timetable solver pipeline (Phase 0–5)

Standing design for [[timemath-project]]. Each phase either produces a
strictly smaller problem for the next, or fails with a specific, named,
actionable reason — an "infeasible" without a named culprit has not done its
job.

| Phase | Does | Status (2026-07-26) |
|-|-|-|
| 0 — Normalise | parse rosters + `TT<year>.xlsx`, apply conversion table, hard-fail unknown codes, derive existence table, expand qualifications, derive baskets, quarantine exceptions | DONE (Excel) |
| 1 — Bounds | compute lower bounds (basket size, clique, load, teacher column demand, Hall/covering), take the max, fail fast | DONE (Excel) |
| 2 — Precolour | pin universal block, rank + pin anchor teacher (sound symmetry break), propagate basket-SDR + teacher-exclusivity to fixpoint, fail with a Hall-certificate name on domain collapse | ACTIVE — coding, Grade 12 2026 |
| 3 — Column + student assignment | `y[student,section]` CP-SAT: student→section, per-student all-different on columns, teacher exclusivity cross-grade, cost = spread beyond floor | EXPLORED only — see [[phase3-column-student-assignment]] |
| 4 — Teacher assignment | bipartite match sections→qualified teachers; minimise distinct teachers, or compute min-cost repair if infeasible | EXPLORED only — see [[phase4-teacher-assignment]] and [[repair-model]] |
| 5 — Cell expansion | expand placed sections into grid cells; reinstate quarantined exception students | EXPLORED only |

Key inversion vs the old (failed) codebases: `Section` is the decision-
carrying entity (not `Lesson`); `Lesson`/`TimeSlot` are *derived output*,
generated only after a `Section` is placed — not the other way round.

Related: [[basket-sdr]] (the binding constraint through Phases 1–3),
[[universal-block-and-band]] (what Phase 2 pins first),
[[independent-verification]] (`ClashDetector`, runs on every Phase 5
output).
