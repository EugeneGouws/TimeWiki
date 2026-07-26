---
tags: [summary, timemath]
created: 2026-07-26
updated: 2026-07-26
sources: [raw/sources/timemath/docs/PRODUCT.md]
---

# Summary: `docs/PRODUCT.md`

Defines "done" for [[timemath-project]]: given students+choices,
teachers+qualification pools, timetable shape, and caps — produce a feasible
timetable with minimal teacher usage, repeatably, for any school/year. Five
prior attempts failed by building before the model was understood; this doc
exists to fix that. Definition of done: CP-SAT formulation returning either
a feasible timetable or a specific actionable failure reason (never silent);
every output passes an independent verifier sharing no solver code (see
[[independent-verification]]); validated on ≥2 real years; the
raw-data→model pipeline is itself repeatable and tested. Lays out the full
Phase 0–5 mental model — see [[timetable-solver-pipeline]] for the table.
Non-goals: UI, deployment, optimising anything beyond teacher count/manual
effort.
