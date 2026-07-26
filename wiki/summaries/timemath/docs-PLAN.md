---
tags: [summary, timemath]
created: 2026-07-26
updated: 2026-07-26
sources: [raw/sources/timemath/docs/PLAN.md]
---

# Summary: `docs/PLAN.md` (repo root)

Build order for [[timemath-project]], companion to PRODUCT.md/CLAUDE.md.
Object model (`Section` is the decision-carrying entity, not `Lesson`).
Phase 0 (normalise, parse+convert+validate+existence table), Phase 1 (bounds
— basket size, clique, load, teacher column demand, Hall/covering; both DONE
and Excel-validated). Phase 2 (precolour — symmetry breaks + domain
reductions only, heuristics go to `AddHint` not constraints; see
[[timetable-solver-pipeline]]). Phase 3 (CP-SAT column assignment — this
copy still shows the **original, superseded** subject-level SDR formulation,
corrected in the solver-docs copy — see [[phase3-column-student-assignment]]
and [[solver-docs-consolidation]]). Phase 4 (teacher matching, minimise
distinct teachers). Phase 5 (cell expansion). §8 verification via
`ClashDetector`. §9 build order, steps 1–4 buildable now, step 5+ needs
teacher-file rebuild.
