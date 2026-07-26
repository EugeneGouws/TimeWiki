---
tags: [entity, timeedusuite, component, live]
created: 2026-07-26
updated: 2026-07-26
sources: [raw/sources/timeedusuite-timebuilder.md]
---

# TimeBuilder

Qt6/C++ console app, part of [[timeedusuite-suite]]. Constructs a full
school timetable from scratch (student-subject + teacher-subject
spreadsheets) into the 8×7 grid.

## Pipeline (six stages, no backtracking between stages)

1. **StudentAllocator** — section assignment + heavy-layer (M=7) block
   colouring.
2. **BlockPlacer (Phase A)** — vector bin-packing, assigns each lesson to a
   block.
3. **SlotPlacer (Phase B)** — bipartite edge colouring, assigns periods
   within a block.
4. **SAplacer (Phase D)** — simulated annealing, escapes local optima (fixed
   seed 42, deterministic).
5. **[[ejector-repair-engine]] (Phase C)** — atomic cell-placement repair,
   bounded-depth chains.
6. **StudentEjector (Phase C2)** — student-relocation repair, marginal gains
   (+0–1 cells measured).

## Current state (Session T.6, 2026-06-08)

Residual = 19 cells on 2025 data (deterministic, zero mismatches); 18 cells
on unverified 2026 sandbox. Pure local search exhausted — further gains need
algorithm changes (extend SA to student allocation, deepen backtracking) or
staffing changes (split sole-teacher heavy subjects).

## Design constraints (by design, not bugs)

No stage backtracks into a previous stage (residual is local-search-bound by
construction). Block H reserved for seniors (PE+LO+MA_b bundle), excluded
from SA/ejector. PE lessons skip teacher-load checks everywhere (three
sites: BlockPlacer, SlotPlacer, `Ejector::isTeacherClash`).

Full detail: `raw/sources/timeedusuite-timebuilder.md`.
