---
tags: [concept, timemath, testing]
created: 2026-07-26
updated: 2026-07-26
sources: [raw/sources/timemath/docs/PRODUCT.md, raw/sources/timemath/solver-docs/PLAN.md]
---

# Independent verification (`ClashDetector`)

**The dangerous failure is a plausible wrong answer, not a crash** — a
timetable returned "feasible" that double-books a teacher, or a validator
that silently reports zeros (see the 2025 grade-type-mismatch bug on
[[timemath-project]]). Golden tests do not catch this class of bug.

`ClashDetector`, inherited in concept only from [[timeedusuite-suite]], must
**share no code with the solver** and runs on every complete timetable, in
tests and in production. If solver and verifier disagree, that's caught
immediately rather than discovered downstream.

Layered with: non-zero-scope guards on every verdict, exhaustive tiny
brute-force instances, infeasibility tests asserting the *named culprit* is
correct (not just that infeasibility was detected), and year-over-year
regression fixtures.
