# TimeEduSuite Core Engine

Shared C++ static library (`timeedu_core`). Linked by TimeBuilder, TimeEditor, and future TimeExam. Holds the suite's reusable spine: domain model, grid/cell model, repair engine, JSON I/O.

**Status:** Live (T.5a–d complete). Domain model, allocator, grid model, ejector, JSON contract all extracted and versioned.

---

## Architecture Overview

**Three language tiers** meeting at a single JSON contract:
- **Python** (headless): Converter (Access/xlsx ↔ `timetable.json`)
- **C++/Qt** (desktop): Builder, Editor, Exam (linked to `core`)
- **React/Vite** (web): TimeView (client-only, static host)

**Single repair spine:** The ejector (Phase C construction repair) is **shared core**, used by every desktop tool in three cases:
1. Construction (bulk defect sweep)
2. Manual edit (single defect + cascade)
3. Rollover (grade advance, bulk re-allocation)

All three invoke the same `IRepairEngine` interface (v1.0, locked 2026-06-02).

---

## Domain Model

**Central hub:** `TimetableModel` (formerly `DataSets`). Entities are first-class, never parsed from labels.

### Data Types

- **Lesson** (`lesson.h`)
  - `name` = opaque unique identifier (never parsed)
  - `subjectCode` = "MA", "EN", "PE" (not composite)
  - `grade`, `Teacher T`, `bitset<512> S` (enrolled students)
  - `M` = multiplicity (periods per cycle, from `getM(code, grade)`)
  - `instanceIndex` = 1+ (for duplicate cohorts, e.g., MA vs MA_b)
  - `cells` = **canonical** `QList<(block, slot)>` — the actual placement

- **Subject** (composite key, e.g., "MA_10")
  - Grade-scoped grouping of sibling lessons (same subject+grade, different teacher/cohort)
  - Matched by equality on name; never split in code

- **Student**, **Teacher** (bitset-indexed)
  - `MAX_STUDENTS = 512`, `MAX_TEACHERS = 64`
  - Fast set algebra via bitsets

- **Block** (8 named A–H)
  - Container for lessons placed in that block
  - 7 periods per block → 56 slots total

### Grid / Cell Model

- **8 blocks × 7 periods = 56 timeslots** per student/week (A1–H7)
- **Grade axis implicit** (students bound to grade; teachers cross-grade via OccupancyIndex)
- **Lesson placement canonical:** `cells = [(block, period), ...]` (size must equal M)
- **Legacy fields** (`placement` block name, `slotMask` periods) derived from `cells`, kept in sync by `resync()`

### Occupancy Index

`OccupancyIndex::teacherBusy[block][period]` = bitset of teachers at that slot across **all grades**. Backbone for cross-grade teacher-clash detection (Phase B avoids creating clashes; Phase C repairs residuals).

### Key Methods

- `internStudentBit(id)`, `internTeacherBit(name)` → bit assignment (ingest API)
- `getLessonMutable(name)` → **asserts pre-existence** (use after `insertLesson`)
- `getLessonsBySubject(code, grade)` → sibling lessons for student rehome/swap
- `promoteReserveLesson()` → uplift from surplus pool (used only by Phase A/B, not ejector)

---

## Repair Engine (Phase C: Ejector)

**Mission:** Place every wedged lesson + resolve residual teacher clashes via bounded ejection chains.

### Key Concepts

- **Wedge** = lesson with `cells.size() < M` (couldn't be placed greedily)
- **Blocker** = placed lesson at a candidate cell, sharing a student (same grade) or teacher (non-PE)
- **Ejection chain** = relocate blocker (recursively) to free the cell
- **Transactional** = all mutations recorded in `Mutation` list; rollback on failure

### Constraints (Hard, Always Preserved)

1. Lesson completeness: every lesson reaches `cells.size() == M`
2. Student-disjoint: no student twice in any (block, period)
3. **Teacher-disjoint (non-PE globally)**: no non-PE teacher in two lessons at same (block, period) across all grades
4. Per-student per-block capacity ≤ 7
5. Per-teacher per-block capacity ≤ 7 (PE exempt)
6. Class-size cap: ≤ 25 (or 50 for PE)
7. Student↔lesson invariant: bitsets and lesson lists stay in sync

### PE Special Rule (Critical)

- PE lessons **exempt** from teacher-clash logic
  - Multiple PE lessons **can** share (block, period) with same teacher
  - Identify by `subjectCode == "PE"`
  - Class cap = 50 (vs 25 for others)
- **Senior bands not split:**
  - MA (M=7) + MA_b (M=3) paired (same teacher, same students)
  - PE + LO bound 1:1 to each senior math class

### Repair Mechanisms

- **Y-axis move** = relocate lesson to another free (block, period)
- **Student rehome** = move shared student to sibling lesson (same subject+grade)
- **Student swap** = exchange student with peer in sibling
- **Recursion** = eject blocker's blocker if needed (depth-bounded)
- **Rollback** = undo entire chain on failure (transactional)

### Budgets (Tunable)

- `EJECTION_NODE_BUDGET = 50000` (default search nodes)
- `EJECTION_PER_TARGET_BUDGET = 500` (per defect)
- `EJECTION_MAX_DEPTH = 16` (chain depth)

### IRepairEngine Contract (v1.0)

**Interface shape (REPAIR_INTERFACE.md v1.0):**

```cpp
// Transactional (single active savepoint per engine)
Savepoint createSavepoint();
void      rollbackTo(Savepoint);
void      commit(Savepoint);

// Detection (pure, no mutation)
QList<Defect> detectDefects() const;  // wedges, clashes, integrity warnings

// Single-defect repair (binary outcome)
RepairResult repair(const Defect&, const RepairBudget& = {});
// → Resolved | FailedNoMove | FailedExceededBudget | NotImplemented

// Bulk repair (rollover, construction wedge sweep)
RepairResult repairAll(const QList<Defect>&, const RepairBudget& = {});
// → Resolved | PartiallyResolved | FailedNoMove | FailedExceededBudget | NotImplemented
```

**Defect types:** StudentClash, TeacherClash, Both, CapacityExceeded, Unwedge, IntegrityWarning

**Two implementations:**
- `StubRepairEngine` (Editor, E.0–E.6): trivial moves only; returns `NotImplemented` for chains
- `EjectorRepairEngine` (core, T.5c): real engine, wraps `Ejector`

**Key invariants:**
- Savepoint is engine-managed (no nesting)
- `repair()` outcome is binary (never `PartiallyResolved`)
- `PartiallyResolved` exclusive to `repairAll()`
- `FailedNoMove`/`FailedExceededBudget` leave cube **byte-identical** to entry (safe undo)

---

## JSON Contract (Schema v3.0)

**File:** `timetable.json` (produced by converter, consumed by Editor/TimeView).

**Version discipline:** Major version bump = breaking change. Consumers declare major version target; mismatch fails loudly at load.

### Top-Level Structure

```json
{
  "version": "3.0",
  "generated_at": "ISO-8601-UTC",
  "source": {"accdb": "2025.accdb", "read_at": "..."},
  "timeslots": ["A1", ..., "H7", "P1", ..., "P4"],
  "students": {"id": {"name", "grade", "reg_class", "gender", "house"}},
  "teachers": {"id": {"surname", "initials", "display_name", "venue"}},
  "lessons": {"label": {"teacher_id", "subject_code", "grade", "student_ids", "timeslots"}},
  "enrolments": {"label": {"student_ids"}},        // derived
  "placements": {"label": {"timeslots"}},         // derived
  "student_slots": {"student_id": {"label": ["timeslot", ...]}},  // normative
  "free_periods": {"students": {...}, "teachers": {...}}
}
```

**Normative sources:** `lessons`, `student_slots` (the ground truth). `enrolments`, `placements` are **derived** and recomputed on every write.

**Lesson label format:** `CODE_SURNAME_GG[_N]`
- `CODE` = subject code
- `SURNAME` = teacher surname (uppercased, spaces→`_`)
- `GG` = grade
- `_N` = optional instance suffix (splits only, e.g., `_2`, `_3`)
- `_GG` suffix marks original; no teacher id allowed (breaks converter round-trip)

**Lossy aspects (v2.1 → v3.0):**
- Builder write omits: gender, house, display_name, venue, free_periods, P-slots
- Passthrough container (`JsonPassthrough`) carries schema fields TimetableModel doesn't interpret
- Converter produces best-effort teacher matching (surname-based)

### Key Design Principles

1. Timeslot is primary axis
2. Curricular + free-period data strictly separated
3. Numeric IDs stable for teachers/students (surnames/codes display-only)
4. `lessons` + `student_slots` normative; `enrolments`/`placements` derived

---

## StudentAllocator

**Input:** Raw lessons (one per teacher+subject+grade offering).

**Output:** Trimmed+ordered lesson set ready for Phase A (BlockPlacer).

**Pipeline:**

1. **Trim lessons** to feasible cohort size (ceil(takers / 22))
2. **Surplus → reserve pool** (never deleted)
3. **Edge-min greedy student assignment** (minimize cost within each grade)
4. **Heavy-layer pre-coloring** (M=7 subjects → dedicated blocks, preserving teacher compactness)
5. **Senior specials:**
   - MA duplicate: MA (M=7) + MA_b (M=3), same students/teacher
   - PE/LO band: each senior math class binds 1:1 to PE + LO

---

## Known Issues & Gotchas

### Active Risks (Re-check Every Session)

1. **Lesson labels must be `CODE_SURNAME_GG`** — never embed teacher ID
   - Converter splits `_SURNAME` back out for ST1/TT1 round-trip
   - Numeric `_2`/`_3` suffixes only for split instances
   - Grep new labels before handing back

2. **`getLessonMutable()` asserts pre-existence**
   - Contract: caller must call `insertLesson()` first
   - Applies to any TimetableModel consumer (JSON load, bridge, test fixtures)
   - Use: `Lesson l; /* populate */; model.insertLesson(l);` **then** `model.getLessonMutable(name)`

3. **Variable `slots` collides with Qt macro** (caught 5× in production)
   - Qt defines `slots` as an empty preprocessor macro for moc
   - Collision in **any** TU including Qt headers, not just QObject classes
   - **Fix:** never name `slots`, `signals`, `emit`, `foreach`. Use `timeSlots`, `slotList`, `slotIds`
   - Grep new C++ for `\bslots\b` before review

4. **Subject codes drift across sources**
   - Same subject spelled differently: ST1="EMS" vs hand-made="EM" vs getM table="EMS"
   - Each mismatch cascades: missing M → inflated load → wedges
   - **Fix:** derive builder inputs from ST1 via `tools/make_builder_inputs.py` (authoritative)
   - Keep `lessoninfo.cpp::getM` aligned to ST1 codes; never hand-maintain student/teacher files

5. **OOT is per-student, not per-code**
   - Out-of-timetable subjects live in P1–P4 per individual student
   - Subject label can have A–H placements for some students, P-only for others
   - **Fix:** strip OOT per student from `student_slots` (their actual A–H placements)
   - Always-OOT codes (FS*, MANDARIN) dropped by name as backstop

6. **Builder was nondeterministic** (T.5 resolved, pattern applies to new executables)
   - Qt6 ignores `qputenv("QT_HASH_SEED")`; reads seed too early
   - **Fix:** first line of `main()`: `QHashSeed::setDeterministicGlobalSeed();`
   - Any "re-run = identical" gate is meaningless without this

### Resolved (Pattern Still Applies)

- **`Subject.M` on student copy stale** → use `getM(code, grade)` always
- **Converter baselines locked** → update them intentionally with behavioural changes
- **HANDOFF.md can lag** → cross-check against CLAUDE.md status + src scan
- **PE/LO band pinning** → ejector never ejects PE or LO cells (immovable hard blockers)

---

## Strategy & Planning Notes

### Regularity Discipline

Compactness baked into construction + repair (Phase A tie-break + repair move ordering + cross-block swap gating), not only Phase D SA:

1. **Phase A tie-break:** prefer blocks keeping teacher's lessons in fewer, consistent blocks
2. **Phase C tie-break:** among equally-feasible repairs, prefer cube-compacting moves
3. **Gate cross-block swap:** the one repair scattering teacher's week — budget/penalise harder

Within-block period moves preserve block-level compactness by construction. Phase D SA targets soft objectives (gaps, spread) once Phase C delivers structural feasibility.

### Build Sequence

1. **Finish Builder v1** (STEP0→STEP3) → proves engine end-to-end, produces first feasible cube
2. **Extract shared core** (T.5a–d) → domain model, grid, ejector, JSON I/O as internal library
3. **Build Editor** on core → verify, manual edit, repair, then rollover (bulk path through same engine)
4. **Defer Builder Phase D (SA)** and **Exam tool** until v1 of Builder + Editor proven

**Rationale:** engine is the long pole; prove it in Builder (most scaffolding), then lift rather than write twice.

### Core Extraction Decisions (Locked)

- **Library, not framework** — exports headers + static lib; consumers link
- **Non-owning pointers** — engine holds `TimetableModel*`, `QList<Block>*` borrowed from caller
- **Qt Core only** — no QXlsx, no Widgets
- **Synchronous** — caller wraps in `QFutureWatcher` if async needed
- **Interface stability** — `IRepairEngine` + `JsonContract` versioned like schema
- **Repair engine wraps ejector via `friend class`** — no move-set leaks into public surface
- **No reserve-promotion in ejector** — student rehome/swap only (Phase A/B use reserve)
- **Tests live in consumers** — core is headerless library

### Known Residuals & Trade-offs

- **Residual defects on live data:** real ST1/TT1 carry ~41 hard violations (inconsistent source cells). Compare against converter baseline, not zero.
- **Deterministic build required:** validation gates (e.g., `[alloc:infeasible]` = 0) meaningless without `QHashSeed::setDeterministicGlobalSeed()`
- **Partial runs distort diagnosis:** first-fit bias mask real causes; always run full dataset before reasoning about placement

---

## Sources

- `E:\TimeEduSuite\CLAUDE.md` — suite coordination, active projects, session protocol
- `E:\TimeEduSuite\core\CLAUDE.md` — core library status, layout, architectural decisions
- `E:\TimeEduSuite\STRATEGY_BRIEF.md` — suite architecture, three tiers, shared spine, build sequence
- `E:\TimeEduSuite\EJECTOR_SPEC.md` — Phase C ejector complete design (grid, data model, constraints, mechanisms, budgets)
- `E:\TimeEduSuite\REPAIR_INTERFACE.md` — `IRepairEngine` v1.0 contract (locked 2026-06-02, stability discipline)
- `E:\TimeEduSuite\SCHEMA.md` v3.0 — JSON contract, lesson label format, normative sources
- `E:\TimeEduSuite\mistakes.md` — suite-level gotchas (active + resolved)
- `E:\TimeEduSuite\core\include\*` — header files (TimetableModel, Lesson, Block, Ejector, EjectorRepairEngine, JsonContract, StudentAllocator, etc.)
