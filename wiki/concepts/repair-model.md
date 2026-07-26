---
tags: [concept, timemath, explored]
created: 2026-07-26
updated: 2026-07-26
sources: [raw/sources/timemath/solver-docs/docs/explorations/repair-model.md]
---

# Repair model — minimum-cost fix on infeasibility

**Status: EXPLORED, NOT IMPLEMENTED.** [[timemath-project]] demands a
specific, actionable failure reason (see [[independent-verification]] /
product definition of done). This goes one further: propose the cheapest
fix, not just name the culprit.

## Approach

Model qualification tokens (and candidate section splits) as CP-SAT
assumption literals; on infeasibility, extract
`sufficient_assumptions_for_infeasibility` to name the deficient set.
**Minimum-cost repair** = fewest additional qualification tokens/splits that
make the instance feasible — e.g. "infeasible unless someone picks up SC_11"
beats "infeasible". Gate with the split/clique pre-pass (cheaper than
exhaustive enumeration).

## Findings

* Exhaustive enumeration of all minimum-cost repair sets took ~91s — cost is
  in enumeration, not in finding one repair. Find one fast; enumerate all
  only on request.
* Combined with [[phase4-teacher-assignment]]'s reframe, the pipeline
  becomes: feasible? done. Infeasible? named culprit + cheapest repair.

## Open

* Cost function: are all qualification tokens equally priced, or weighted by
  scarcity?
* A repair should never be proposed for a problem an exception student
  (quarantined basket) caused — interaction not yet resolved.
