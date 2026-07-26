---
tags: [entity, timeedusuite, component, live]
created: 2026-07-26
updated: 2026-07-26
sources: [raw/sources/timeedusuite-timeeditor.md]
---

# TimeEditor

Qt6/C++ desktop app, part of [[timeedusuite-suite]]. Views, edits, and rolls
over timetables. Replaces the archived [[timeverify]] and manual rollover
scripts. Reads/writes `timetable.json` ([[json-timetable-schema]]).

## Architecture

Strict layering: `ui/` (Qt widgets only) → `io/` (JSON via core
`JsonContract`) → `verify/` (clash/integrity, no Qt) → `edits/` (edit-stack
undo/redo) → `rollover/` (S0–S8 pipeline) → `repair/` (bridges to core's
[[ejector-repair-engine]] via `IRepairEngine`). `verify/` and `edits/` never
import Qt widgets — kept portable/testable.

## Current state (LC.3, 2026-06-29)

Pivoted from a general editor (E.0–E.7, project goal residual=0, done) to a
**learner-change tool** (LC track): ACCDB import/export, rule-driven
subject/teacher swaps (MATHS, FAL, COMPULSORY, OPTION, ELECTIVE rules),
targeted reshuffle, cohort-aware swap filtering.

**Open gap:** LC.4 (extra-lesson/partial-attendance handling) reverted —
detection rule was wrong, needs re-derivation before retry.

## Rollover pipeline (S0–S8)

Implements [[rollover-year-advancement]]: load/clear (S0–S1) → advance
grades (S2) → prune graduating cohort (S3) → apply subject choices + ingest
new grade-8s (S4) → snapshot defects (S5–S6) →
`EjectorRepairEngine::repairAll()` (S7, depth ≤3, budget 200k nodes) →
export with provenance (S8). This is the C++ successor to the standalone
[[rollover-cpp-prototype]] — same problem, far more complete clash handling
(routes through the shared ejector instead of simple augmenting-path
matching).

## Known gotchas

ACCDB import/export shells out to Python converter scripts via `QProcess` —
deployment must bundle Python 3 + ACE OLEDB. Lesson labels never minted with
teacher ID (see [[timeedusuite-core-library]] gotchas). No identifier named
`slots` (Qt moc collision).

Full detail: `raw/sources/timeedusuite-timeeditor.md`.
