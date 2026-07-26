---
tags: [concept, timeedusuite, data-format]
created: 2026-07-26
updated: 2026-07-26
sources: [raw/sources/timeedusuite-core.md, raw/sources/timeedusuite-tools-and-archive.md]
---

# `timetable.json` schema (v2.1 / v3.0)

The single contract all three tiers of [[timeedusuite-suite]] meet at —
produced by [[converter-toolchain]], consumed by [[timebuilder]],
[[timeeditor]], [[timeview]]. Version discipline: major bump = breaking
change, consumers declare target version, mismatch fails loudly at load (not
silently).

## Structure

Top-level: `version`, `timeslots` (A1–H7 + P1–P4 out-of-timetable),
`students`, `teachers`, `lessons`, plus **derived** sections recomputed on
every write: `enrolments`, `placements`. Normative (ground truth) sources
are `lessons` and `student_slots`.

## Lesson label format: `CODE_SURNAME_GG[_N]`

`CODE` = subject code, `SURNAME` = teacher surname, `GG` = grade, optional
`_N` = instance suffix for split sections. **Never embed a teacher ID in the
label** — this breaks the converter's ST1/TT1 round-trip. This convention is
a recurring source of bugs when violated (see [[timeedusuite-core-library]]
gotchas).

## `instanceIndex`: the field that killed string-parsing

Originally, code across the suite parsed lesson/subject name strings
(`endsWith("_b")`, `startsWith(...)`) to recover semantic meaning — brittle,
and it blocked shared-core extraction (the repair engine contract forbids
label parsing). Fixed (T.0/`LESSON_REFACTOR.md`) by adding real fields:
`Subject.code`, `Lesson.subjectCode`, `Lesson.instanceIndex`. The same field
is used independently by [[converter-toolchain]] (C.1) to disambiguate
duplicate teacher offerings in JSON and by [[timebuilder]] (T.0) to avoid
re-parsing names — a single shared field solving two problems, a useful
pattern: **when two components both need to disambiguate duplicates, give
them one shared identity field, not two parsers of the same string.**

## Design principles

Timeslot is the primary axis; curricular and free-period data are strictly
separated; numeric IDs are stable for students/teachers (surnames/codes are
display-only, never used as keys); `lessons` + `student_slots` are
normative, everything else is derived and must never drift from them.

## Known limitation

Write-back is ST1-only — TT1 (teacher grid) isn't reconstructable from JSON
and is left as-is, re-derived on next read. A v2.2 schema bump would be
needed if team-teaching (>1 teacher per label) becomes a real requirement.
