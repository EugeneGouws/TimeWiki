---
tags: [entity, timeedusuite, component, live]
created: 2026-07-26
updated: 2026-07-26
sources: [raw/sources/timeedusuite-core.md]
---

# TimeEduSuite core

Shared C++ static library (`timeedu_core`), part of [[timeedusuite-suite]].
Linked by [[timebuilder]], [[timeeditor]], and future TimeExam. Status:
live, T.5a–d complete.

## What it owns

- **Domain model** — `TimetableModel` hub.
  Lesson/Subject/Student/Teacher/Block as first-class entities, never parsed
  from label strings (see [[json-timetable-schema]] for why that matters).
- **Grid model** — 8 blocks (A–H) × 7 periods = 56 timeslots/week.
  `OccupancyIndex` tracks cross-grade teacher busy-ness.
- **[[ejector-repair-engine]]** — the shared repair spine, exposed via
  `IRepairEngine` v1.0 (locked 2026-06-02).
- **JSON I/O** — `JsonContract`, see [[json-timetable-schema]].
- **StudentAllocator** — pre-placement stage: trims lessons to feasible
  cohort size, edge-min greedy assignment, heavy-layer (M=7) pre-colouring,
  senior MA/PE/LO band pairing.

## Extraction decisions (locked)

Library not framework (consumers link, non-owning pointers into caller's
`TimetableModel`), Qt Core only (no Widgets/QXlsx), synchronous, tests live
in consumers. Rationale documented in [[python-prototype-to-cpp-pattern]]
and the suite's "prove in Builder, then lift" build sequence — build Builder
v1 first (proves the engine end-to-end), extract core, build Editor on top,
defer Exam and SA tuning until both are proven.

## Known gotchas (active, re-check each session)

- Lesson labels must be `CODE_SURNAME_GG[_N]` — never embed teacher ID in
  the label; converter round-trip depends on this.
- `getLessonMutable()` asserts pre-existence — must `insertLesson()` first.
- Variable name `slots` collides with the Qt moc macro in any translation
  unit that includes Qt headers (caught 5× in production) — use
  `timeSlots`/`slotList` instead.
- Subject codes drift across sources (ST1 vs hand-made vs getM table) —
  derive from ST1 via `tools/make_builder_inputs.py`, never hand-maintain.
- OOT (out-of-timetable) is per-student, not per-subject-code.
- Builder was nondeterministic until
  `QHashSeed::setDeterministicGlobalSeed()` added as first line of `main()`
  — Qt6 ignores the `QT_HASH_SEED` env var if read too late. Applies to any
  new executable in the suite.

Full detail: `raw/sources/timeedusuite-core.md`.
