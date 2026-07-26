# TimeBuilder — School Timetable Generator

## Purpose

Qt6/C++ console application that constructs a complete school timetable from two input spreadsheets: which students take which subjects, and which teachers teach which subjects. Generates a deterministic 8-block × 7-period weekly schedule (56 cells total) with global teacher availability constraints and per-student load balancing.

## Architecture

Six-stage greedy pipeline with no backtracking between stages:

1. **StudentAllocator (Stage 1)** — assigns students to lesson sections and block-colours senior M=7 subjects
   - Minimizes student-conflict-graph weighted edge cost (weight = lesson multiplicity M)
   - Senior heavy subjects (M=7, grades 10–12) constructively colour-assigned to blocks A–G with depth-2 backtracking
   - Juniors assigned by register class + edge-min greedy + hill-climb local search
   - Creates sibling math sections (MA_b) and assigns PE/LO 1:1 to math cohorts

2. **BlockPlacer (Phase A)** — assigns each lesson to one block
   - Vector bin-packing: respects both student capacity (∑M ≤ 7 per block) and teacher capacity (∑M ≤ 7 per block)
   - Three passes: seed pre-allocated (seniors from Stage 1), DSATUR heavy (M=7), least-loaded-fit light (M<7)
   - PE lessons skip teacher-load checks (mass supervision model)

3. **SlotPlacer (Phase B)** — assigns periods within the lesson's block
   - Bipartite edge colouring: maximal student-disjoint, teacher-free sets per period
   - Seven greedy passes (one per period), hardest-first within each pass
   - Wedged lessons (remaining > 0) left unplaced for ejector

4. **SAplacer (Phase D)** — simulated annealing escape from local optimum
   - Moves: swap two lessons between different blocks (excluding H), re-colour affected blocks, accept via Metropolis
   - Cost: sum of per-lesson cell deficit
   - Fixed seed (42) for determinism; cooling T: 3.0 → ~0.01 over 100k iterations or early exit at 20k no-improve

5. **Ejector (Phase C)** — atomic cell-placement repair via bounded-depth chains
   - For each lesson short a cell: enumerate blocker-count per candidate cell, progressively escalate depth
   - Clear blockers structurally (student relocation, cell y-axis moves) or eject-and-replace recursively (depth ≤ 3)
   - Transaction log with rollback; global node budget 150k, per-target 500

6. **StudentEjector (Phase C2)** — student reassignment repair (separate from Ejector)
   - Relocates clash-causing students to sibling sections (depth ≤ 3)
   - Skips teacher clashes (C doesn't)
   - Measured +0–1 cells on current data; marginal gains

## Current State

**Session T.6 (Phase D simulated annealing), completed 2026-06-08:**
- **2025 inputs verified:** residual = 19 cells (deterministic, zero student_lesson_mismatches)
- **2026 inputs sandbox:** residual = 18 cells (unverified, needs rebuild with main.cpp repointed)
- Export via `--export-json` validates to schema v2.1 with zero hard errors
- All tunable depths/budgets centralized in `core/include/constants.h`

**Completed phases:**
- Stage 1 (StudentAllocator with heavy-layer depth-2 backtracking): DONE
- Phase A (BlockPlacer three-pass): DONE
- Phase B (SlotPlacer seven-pass): DONE
- Phase D (SA block swaps, pre-ejector): DONE
- Phase C (Ejector cell-displacement): DONE
- Phase C2 (StudentEjector student-relocation): DONE

**Next session goal:** Residual reduction requires algorithm changes (extend SA to student allocation, anneal StudentAllocator hill-climb, deepen backtracking) or staffing decisions (split sole-teacher heavy subjects). Pure local search exhausted.

## Known Issues & Design Decisions

**Architectural constraints (by design):**
- No stage backtracks into previous stages → residual is local-search-bound
- Block H reserved for seniors (PE+LO+MA_b bundle); excludes SA block swaps and ejector perturbations
- SA runs pre-ejector (cells must be single-block; post-ejector invariant broken)
- PE lessons skip teacher-load checks (appears in exactly three sites: BlockPlacer, SlotPlacer, Ejector::isTeacherClash)

**Known gotchas:**
- `Subject.M` on a student's subject copy is stale (0); always call `getM(code, grade)`
- Two M sources require manual sync: `core/src/lessoninfo.cpp` (C++) and `tools/converter/subjectData.py` (Python)
- Class-cap inconsistency: `StudentAllocator::placeStudent` hard-codes 26; ejectors use `classCapFor(subjectName)` returning 100 for PE
- `OccupancyIndex` becomes stale after Phase C; nothing reads it afterwards (harmless today)

**Determinism hardened:**
- `QHashSeed::setDeterministicGlobalSeed()` in main
- RNG seed 42 in SA
- Name-based tiebreakers in all sorts
- Two runs = identical results (regression gate enabled)

**Residual diagnosis (19 cells on 2025):**
- Structural causes: local-search-bound greedy + non-backtracking pipeline
- Likely types: teacher-clash-bound (needs lesson cell moves C can't achieve at depth limit) or single-section-heavy (sole-teacher M=7 subject, no siblings to eject to)

## Sources

- `E:\TimeEduSuite\TimeBuilder\ARCHITECTURE_TOUR.md` — comprehensive design walkthrough
- `E:\TimeEduSuite\TimeBuilder\HANDOFF.md` — session T.6 completion, build verification protocol
- `E:\TimeEduSuite\TimeBuilder\README.md` — project overview
- `E:\TimeEduSuite\TimeBuilder\main.cpp` — 106-line driver orchestrating all stages
- `E:\TimeEduSuite\core\src\StudentAllocator.cpp` — Stage 1 (848 lines)
- `E:\TimeEduSuite\core\src\Ejector.cpp` — Phase C (558 lines)
- `E:\TimeEduSuite\core\src\StudentEjector.cpp` — Phase C2 (347 lines)
- `E:\TimeEduSuite\TimeBuilder\blockplacer.cpp` — Phase A (176 lines)
- `E:\TimeEduSuite\TimeBuilder\slotplacer.cpp` — Phase B (95 lines)
- `E:\TimeEduSuite\TimeBuilder\saplacer.cpp` — Phase D (184 lines)
- `E:\TimeEduSuite\core\include\constants.h` — all tunable parameters
