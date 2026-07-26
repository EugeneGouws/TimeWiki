---
tags: [concept, timeedusuite, domain-logic]
created: 2026-07-26
updated: 2026-07-26
sources: [raw/sources/timeedusuite-timeeditor.md, raw/sources/rollover.md]
---

# Rollover (year advancement)

The domain problem: carry a school timetable forward from one academic year
to the next — advance every student's grade, remap subject offerings,
re-detect clashes created by the shuffle. Solved twice: once in the
standalone [[rollover-cpp-prototype]] (simple, Access-only), once properly
in [[timeeditor]]'s S0–S8 pipeline (uses [[ejector-repair-engine]] for
defect resolution).

## Grade advancement is not `grade += 1`

South African high school runs two-year subject cycles across grades 8–12.
Advancement rule: Gr 8/10/11 advance normally; **Gr 9 resets to 8** (failed
students re-take, subject choices change); **Gr 12 moves to 10**
(post-graduation grade compression — a "continuant" concept specific to this
school's structure, not standard grade progression). Both implementations
encode this identically — worth remembering as a fixed domain rule, not a
tunable.

## TimeEditor's S0–S8 pipeline (the complete version)

S0–S1: load, schema check, clear free periods. S2: advance grades (rule
above), remap groups. S3: remove graduating cohort, prune empty subjects.
S4a–b: apply grade-10 subject choices, ingest new grade-8 cohort. S5–S6:
snapshot defects (StudentClash, TeacherClash) via comprehensive 3D
cross-grade checking. S7: `EjectorRepairEngine::repairAll()` — bulk defect
resolution, depth ≤3, budget 200k nodes / 5k per target. S8: export with
rollover provenance (marks `draft=true` if residual defects remain).

## What the standalone prototype did instead

Detected clashes (teacher/student/venue via bitset overlap) but did not
*repair* them — reported unallocated lessons as diagnostics only. Teacher
assignment used Kuhn's bipartite matching (see [[rollover-cpp-prototype]])
rather than the ejector. Good enough for a proof of concept; the gap (no
repair, no JSON export) is exactly what TimeEditor's version closes.
