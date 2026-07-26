---
tags: [entity, timeedusuite, legacy-attempt, archived, superseded]
created: 2026-07-26
updated: 2026-07-26
sources: [raw/sources/timeedusuite-archived.md]
---

# TimeVerify

Archived attempt (`E:\TimeEduSuite\Archived\TimeVerify`), part of
[[timeedusuite-suite]] history. School timetable verification tool: clash
detection (double-bookings, out-of-subject teacher assignments), interactive
edit-with-undo, JSON export for downstream tools.

## Approach

Python 3.13 + Tkinter, layered (pure verify → app → file_io → UI). Two
verification passes (student double-booking, teacher qualification).
Edit-stack pattern for undo/redo — this specific pattern is the part judged
worth keeping.

## Why abandoned (2026-06-02)

**Fully superseded**, not performance-blocked like [[timepybling]] —
[[timeeditor]] (C++/Qt) directly ported `clash_checker.py`,
`integrity_checker.py`, and the edit-stack pattern
(`ChangeStudentSubject`/`MovePlacement` edits). Preserved only as a port
reference / CLI-parity oracle for testing the C++ version.

## Verdict on the UX pattern

The "verification-first" workflow (detect clashes → interactive fix with
undo) was judged correct and carried forward unchanged into TimeEditor's
`verify/` and `edits/` layers — the lesson here is about tech stack, not
design.

Full detail: `raw/sources/timeedusuite-archived.md`.
