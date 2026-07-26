# TimeEditor — Timetable Viewer & Learner-Change Manager

## Purpose

Qt6/C++ desktop application for viewing, editing, and rolling over South African school timetables. Reads/writes `timetable.json` (schema v2.1 → v3.0 in LC track); presents an interactive 8-block × 7-period grid with entity selectors (school-wide, student, teacher, subject). Supports drag-drop edits, rule-driven subject/teacher changes, and automated rollover pipeline (S0–S8). All repairs route through `IRepairEngine` (core ejector).

**Replaces:** Python/Tkinter `TimeVerify/` + manual rollover scripts.

## Architecture

Layered design, strict separation of concerns:

```
src/
├── ui/         Qt widgets only (MainWindow, GridWidget, SidePanels)
├── io/         JSON I/O via core::JsonContract (JsonLoader, JsonWriter)
├── verify/     Clash + integrity checks (ClashChecker, IntegrityChecker)
├── edits/      Edit stack patterns (MovePlacement, ChangeStudentSubject)
├── rollover/   S0–S8 pipeline per ROLLOVER_SPEC.md
├── repair/     BridgedRepairEngine wrapping core EjectorRepairEngine
└── main.cpp    Driver (no business logic)
```

**Hard rules:**
- `verify/` and `edits/` never import Qt widgets
- `ui/` accesses TimetableModel only through `io/` and `verify/` adapters
- `repair/` code only calls `IRepairEngine` interface (no internals leak)

**Contracts:**
- `core/IRepairEngine` v1.0 — engine abstraction (stub → real ejector at E.7)
- `core/JsonContract` v2.1/v3.0 — schema I/O
- `io::TimetableDoc` — in-memory document model (independent of core)

## Current State

**Last activity:** LC.3 (learner-change focus), completed 2026-06-29.

**Sessions completed:**
- **E.0–E.7:** Full buildout from scaffold to real-engine swap (project goal: residual=0)
- **LC.1–LC.3:** Pivot to learner-change tool: ACCDB import/export, rule-driven subject/teacher swaps, targeted reshuffle dialog, cohort-coherence swap filtering, two-phase apply

**Live features (LC track):**
- Drag-drop lesson moves (single-label cells only)
- Subject change with rules: MATHS(MA↔ML), FAL(AF↔ZU), COMPULSORY(EN/LO/PE) teacher-change-only, OPTION(Gr8–9), ELECTIVE(Gr10–12)
- Targeted reshuffle: re-sections a student's other subjects without touching placements
- Cohort-aware swap filter: destination must have identical cohort in every A–H slot (uniform student sets)
- CLI modes: `--verify`, `--roundtrip`, `--edit-selftest`, `--bridge-selftest`, `--rollover`

**Current gap:** LC.4 (extra-lesson handling) reverted. Detection rule was wrong (partial-attendance mislabelled rows). Requires rework before re-attempting.

## Algorithm & Design Decisions

**Rollover pipeline (S0–S8):**
- S0–S1: Load, schema check, clear free periods, remove non-curricular subjects
- S2: Advance grades, remap groups to new grade
- S3: Remove graduating cohort, prune empty subjects
- S4a–S4b: Apply grade-10 subject choices, ingest new grade-8 cohort
- S5–S6: Snapshot defects (StudentClash, TeacherClash)
- S7: Run `EjectorRepairEngine::repairAll()` (Phase C cell-displacement, depth ≤ 3, budget 200k nodes, 5k per target)
- S8: Export with rollover provenance (draft=true if residual > 0)

**Edit-stack pattern:**
- Snapshot undo: pre-edit doc state saved, edits are imperative mutations
- Savepoint/commit/rollback via `EditController`
- `--edit-selftest` verifies byte-identical undo/revert

**Doc↔Model bridge:**
- `io::TimetableDoc` (UI-facing) ↔ `core::TimetableModel` (engine-facing)
- Decomposes fused subject labels into sections (maximal subsets with identical student sets)
- OOT students (P-slot only) held out and restored on project-back
- `--bridge-selftest` confirms round-trip loss-free

**Clash detection:** Per-timeslot entity disjointness (student + teacher), using `student_slots` as authoritative for attendance. Red-overlay live clashes, orange-flash rejected drags.

**Grid rendering:** QTableView + custom delegate. Non-matching cells clean-empty (not dimmed). Entity selectors: School-wide (view-only, ambiguous), Student (single label per cell, draggable), Teacher (all taught labels, draggable), Subject (view-only).

## Known Issues

**Reverted features:**
- **LC.4 extra-lesson handling** — partial-attendance detection rule was wrong. Needs re-derive before rework. Eugene investigating.

**Live gotchas:**
- No identifier named `slots` (Qt moc macro conflict); use `timeSlots`
- ACCDB import/export shells to Python scripts (`tools/converter/*.py`) via QProcess; deployment must bundle Python 3 + ACE OLEDB
- Lesson labels `CODE_SURNAME_GG` — never mint with id (converter parses surname); swaps never mint labels
- Rollover S4 cohort source TBD until provided (currently hard-coded)
- Schema v3.0 refinements pending: reg advance, 9→10 phase boundary, surplus-teacher repurpose (same-code/same-phase)

**Design constraints:**
- `defectSurfaced` signal now never emitted (targeted dialog reports its own failures); harmless
- Drag-drop restricted to single-label cells (multi-label ambiguous)
- Swaps cohort-aware: destination section must have identical A–H cohort (uniform student sets)

**Data quality warnings (not bugs):**
- `[mult_mismatch]` — placed cell count ≠ multiple of M (dirty source)
- `[sparse_teacher]` — teachers with < 3 subjects (rollover TBD teachers)
- `[subject_no_teacher]` — labels with no teacher ID (TBD classes, dirty accdb)

## Sources

- `E:\TimeEduSuite\TimeEditor\CLAUDE.md` — project guide, layer rules, architectural decisions
- `E:\TimeEduSuite\TimeEditor\HANDOFF.md` — session LC.3 completion, next goals, gotchas
- `E:\TimeEduSuite\TimeEditor\PLAN.md` — full session sequence (E.0–E.7 + LC.1–LC.4), gates, source-to-port reference
- `E:\TimeEduSuite\TimeEditor\USERMANUAL.md` — user-facing CLI/GUI reference with complete examples
- `E:\TimeEduSuite\TimeEditor\src\main.cpp` — driver entry point (CLI branch detection, event loop setup)
- `E:\TimeEduSuite\TimeEditor\src\io\` — JSON loading/writing (JsonLoader, JsonWriter)
- `E:\TimeEduSuite\TimeEditor\src\verify\` — clash/integrity checks (ClashChecker, IntegrityChecker)
- `E:\TimeEduSuite\TimeEditor\src\edits\` — edit stack patterns (EditStack, MovePlacement, ChangeStudentSubject, SubjectOptions)
- `E:\TimeEduSuite\TimeEditor\src\rollover\` — S0–S8 pipeline
- `E:\TimeEduSuite\TimeEditor\src\repair\` — BridgedRepairEngine, EngineFactory
- `E:\TimeEduSuite\REPAIR_INTERFACE.md` — IRepairEngine v1.0 contract
- `E:\TimeEduSuite\ROLLOVER_SPEC.md` — S0–S8 specification (now standalone tool scope)
- `E:\TimeEduSuite\SCHEMA.md` — JSON contract v2.1/v3.0
