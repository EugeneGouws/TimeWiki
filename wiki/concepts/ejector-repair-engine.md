---
tags: [concept, timeedusuite, algorithm]
created: 2026-07-26
updated: 2026-07-26
sources: [raw/sources/timeedusuite-core.md, raw/sources/timeedusuite-timebuilder.md, raw/sources/timeedusuite-tools-and-archive.md]
---

# Ejector / repair engine (Phase C)

The shared repair spine of [[timeedusuite-suite]], living in
[[timeedusuite-core-library]]. One engine, three call sites: construction
([[timebuilder]] bulk defect sweep), manual edit ([[timeeditor]] single
defect + cascade), rollover ([[timeeditor]] bulk re-allocation, see
[[rollover-year-advancement]]). Exposed via `IRepairEngine` v1.0 (locked
2026-06-02).

## Core idea: monotone Φ

Define Φ = total unplaced cells = Σ(M − cells.size()) over all lessons. Each
ejector transaction is atomic: commit ⇒ Φ decreases by ≥1, rollback ⇒ Φ
unchanged. Φ is bounded below by 0 and starts finite ⇒ termination is
guaranteed. This reframes what looks like a combinatorial explosion (~100
wedged lessons) as ~100 independent, individually-bounded steps — the
insight that made the ejector tractable to design (from `EJECTOR_PLAN.md`,
T.4).

## Mechanism

A **wedge** is a lesson with `cells.size() < M`. To place it, find a
candidate cell; if occupied, the occupant is a **blocker** (shares a student
or non-PE teacher). Try in order: free placement → structural clearing
(Y-axis move, student rehome/swap, no recursion) → recursive ejection (eject
the blocker, re-place it deeper, depth-bounded ≤3). Whole chain is one
transaction — commit or rollback, never partial.
`FailedNoMove`/`FailedExceededBudget` leave the model byte-identical to
entry.

## Budgets (tunable, `constants.h`)

`EJECTION_NODE_BUDGET` (~50k–150k depending on caller),
`EJECTION_PER_TARGET_BUDGET` (~500), `EJECTION_MAX_DEPTH` (3–16 depending on
caller). Arithmetic check used during design: 100 wedges × 500/target ≈
50,000 global budget.

## Special-case rule: PE exemption

PE lessons are exempt from teacher-clash logic (multiple PE lessons *can*
share a slot with the same teacher — mass supervision model) and have a
higher class cap (50 vs 25). Implemented via a single `isTeacherClash`
chokepoint (design decision D2) — a locked rule, not a bug, and easy to
accidentally violate if a new check bypasses that chokepoint.

## Two implementations

`StubRepairEngine` ([[timeeditor]] early sessions, trivial moves only) and
`EjectorRepairEngine` ([[timeedusuite-core-library]], the real engine). Same
interface, swappable — this is what let Editor ship incrementally before the
real engine existed.

Compare to [[rollover-cpp-prototype]]'s teacher-allocation approach (Kuhn's
bipartite matching) — a different technique for a related but distinct
problem (assigning teachers vs. placing cells in a grid).
