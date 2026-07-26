---
tags: [entity, timeedusuite, component, live]
created: 2026-07-26
updated: 2026-07-26
sources: [raw/sources/timeedusuite-tools-and-archive.md]
---

# Converter toolchain

Python scripts (`E:\TimeEduSuite\tools\converter`), part of
[[timeedusuite-suite]]. The data-pipeline bridge: Access `.accdb` (school's
source of truth) ↔ Excel `.xlsx` (hand-edit intermediate) ↔ `timetable.json`
(canonical exchange format, see [[json-timetable-schema]]).

## Scripts

- `timetable_to_json.py` — canonical xlsx→JSON producer, includes the C.1
  lesson-fusion fix (below).
- `accdb_to_json.py` — Access→JSON via PowerShell OleDb, reuses
  `timetable_to_json` internals.
- `json_to_accdb.py` — JSON→Access write-back, never mutates source, has
  `--verify` round-trip check.
- `validate.py` — schema v3.0 validator (hard invariants block export, soft
  checks warn).
- `subjectData.py` — subject code → grade → multiplicity (`M`) lookup,
  shared with [[timeedusuite-core-library]]'s C++ side (`lessoninfo.cpp`) —
  the two must be kept in manual sync, a recurring gotcha.

**Vendored, not copied:** `timetable_to_json.py` ships identically into
TimeView, TimeEditor, and the archived TimeVerify. Fix bugs here, once —
vendoring means a bug fix must ship to all copies simultaneously.

## The lesson-fusion bug (C.1, resolved 2026-06-02)

When one teacher taught two sections of the same subject/grade, the
converter fused both into one `M=14` lesson instead of two `M=7` lessons —
unplaceable (exceeds the 7-period block) and produced phantom clashes
between cohorts that never actually met. Fixed via unique lesson keying
`(teacher, subject, grade, instance)` + a disjoint-cohort split rule + a
loud self-check (refuse `M>7`). The `instanceIndex` field this introduced is
now shared with the Builder side — see [[json-timetable-schema]].

## Other durable lessons

Surname collisions need multi-signal tie-break (code → grade → TT1
slot-overlap → venue → display_name); round-trip gate checks JSON equality
not byte-identical cell text; write-back is ST1-only (TT1 is derived, not
reconstructable).

Full detail: `raw/sources/timeedusuite-tools-and-archive.md`.
