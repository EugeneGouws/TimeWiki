---
tags: [concept, timemath, explored]
created: 2026-07-26
updated: 2026-07-26
sources: [raw/sources/timemath/docs/BRAINSTORM.md, raw/sources/timemath/solver-docs/docs/explorations/bounds-ladder.md]
---

# Bounds ladder — minimum teacher count

**Status: EXPLORED, NOT IMPLEMENTED.** Original question from the earliest
brainstorm phase of [[timemath-project]] (before Phase 0–5 existed as a
name): given N students with free subject choice, how few teachers can staff
the year, and what structural property drives the number?

## Measured ladder (2026: 466 students, cap 25, 56 cells)

| Bound | Value | Meaning |
|-|-|-|
| Concurrency floor | ≈19–20 | teachers on duty at peak, from true-group reconstruction |
| Pooled load bound | ≈26 (at load 41) | total teaching cells / max load, pooling all qualifications |
| Subject-silo bound | ≈48 (at load 41) | same, but each teacher locked to one subject |
| Actual (2026 staffing) | ~55 | — |

## Finding

**Teacher count is driven by subject fragmentation / qualification
structure, not by conflict-graph colouring.** Concurrency is already
near-optimal (mean 21.6 busy vs floor ~19); the 26→48 gap is entirely
qualification silos — i.e. the lever is broadening what teachers are
qualified to teach, not scheduling cleverness.

Unreconciled: a crude load anchor on 2026 rollover data gave 253 minimum
sections at cap 25 vs 190 sections actually run — resolve before trusting
any bound built on 253.

## Open

* When is max-of-bounds tight (achievable)?
* Where basket diversity enters — feasibility only, or the count itself?
* Is the marginal-impact greedy for distributing leftover teachers (after
  the minimum is staffed) provably near-optimal, or arbitrarily bad? Same
  open question recorded in [[timemath-project]].
