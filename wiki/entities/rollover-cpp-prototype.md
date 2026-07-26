---
tags: [entity, timeedusuite, legacy-attempt, superseded]
created: 2026-07-26
updated: 2026-07-26
sources: [raw/sources/rollover.md]
---

# Rollover (standalone C++ prototype)

`E:\Rollover` — a separate, earlier standalone prototype, **not part of**
[[timeedusuite-suite]]'s repo. Minimalist Qt6/C++ CLI tool that advances a
school timetable one year: increments grades, re-allocates teachers, detects
clashes. Works directly against MS Access, no JSON export.

## Why it matters

Proof-of-concept for the rollover problem before [[timeeditor]] built its
far more complete S0–S8 pipeline ([[rollover-year-advancement]]).
Superseded/absorbed, not actively developed — but the core algorithmic idea
(bipartite matching for teacher allocation) is a distinct, reusable
technique worth keeping separate from the ejector-based repair approach
TimeEditor uses.

## Teacher allocation: Kuhn's algorithm

Maximum bipartite matching via backtracking augmenting paths — MRV heuristic
(fewest-option lessons first), grade-match preferred over subject-match
fallback, load-balancing sort within domain, recursive `tryAssign()` bumps
an already-assigned teacher to free capacity. This is a different technique
from TimeBuilder's StudentAllocator (edge-min greedy) or the ejector's
cell-displacement chains — worth remembering as a third option if
teacher-assignment (not cell-placement) becomes the bottleneck elsewhere.

## Grade advancement quirk (South African-specific, reusable knowledge)

Not a simple `grade += 1`: Gr 9 resets to 8 (re-take cohort), Gr 12 moves to
10 (post-graduation grade compression, since SA high school runs two-year
subject cycles on grades 8–12). [[timeeditor]]'s S2 stage implements the
same logic.

## What TimeEditor's rollover does that this doesn't

Comprehensive 3D cube clash checking (cross-grade), integration with
[[ejector-repair-engine]] for actual defect *resolution* (this tool only
detects and reports), JSON export to schema v3.0.

Full detail: `raw/sources/rollover.md`.
