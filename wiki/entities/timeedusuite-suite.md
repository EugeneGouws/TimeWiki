---
tags: [entity, timeedusuite, suite]
created: 2026-07-26
updated: 2026-07-26
sources: [raw/sources/timeedusuite-core.md, raw/sources/timeedusuite-timebuilder.md, raw/sources/timeedusuite-timeeditor.md, raw/sources/timeedusuite-timeview.md, raw/sources/timeedusuite-tools-and-archive.md, raw/sources/timeedusuite-archived.md]
---

# TimeEduSuite

Umbrella project (`E:\TimeEduSuite`) for South African school timetabling:
build, edit, roll over, and view timetables. Hub page — drill into
individual components below.

## Three-tier architecture

- **Python (headless):** [[converter-toolchain]] — Access/xlsx ↔
  `timetable.json`
- **C++/Qt (desktop):** [[timebuilder]] (construct), [[timeeditor]] (edit +
  rollover), TimeExam (planned, C++ port of [[timepybling]])
- **React/Vite (web):** [[timeview]] — read-only viewer, client-only, stable
  and **live in production**

All tiers meet at one contract: [[json-timetable-schema]].

## Shared spine

[[timeedusuite-core-library]] is the C++ static library linked by Builder
and Editor (and future Exam). It owns the domain model, grid/cell model, and
the [[ejector-repair-engine]] — used identically for construction, manual
edit, and rollover. This "one repair engine, three call sites" design is the
suite's central architectural bet.

## Component status (as of 2026-07-26)

| Component | Status |
|---|---|
| [[timeedusuite-core-library]] | Live, T.5a–d complete |
| [[timebuilder]] | Phase D done; residual=19 cells on 2025 data, local-search-bound |
| [[timeeditor]] | LC.3 shipped; LC.4 (extra-lesson) reverted, awaiting rework |
| [[timeview]] | **Stable, live in production** (v1 shipped, pilot 2026-07) |
| [[converter-toolchain]] | Live, vendored into TimeView/TimeEditor/TimeVerify |
| [[timepybling]] | Archived 2026-06-02 — Python perf-blocked, superseding C++ tool (TimeExam) not yet built |
| [[timeverify]] | Archived 2026-06-02 — fully superseded by TimeEditor |
| Rollover (standalone) | Separate earlier prototype, see [[rollover-cpp-prototype]] — absorbed into TimeEditor's S0–S8 pipeline |

## Recurring pattern

See [[python-prototype-to-cpp-pattern]] — TimePyBling→TimeExam,
TimeVerify→TimeEditor, and the standalone Rollover→TimeEditor rollover
pipeline all follow the same shape: Python/Tkinter prototype proves the
UX/algorithm, then gets rewritten in C++/Qt on the shared core once
performance or integration demands it.

## Relationship to TimeMath

`E:\timemath` ([[timemath-project]]) is a separate, currently-active build —
a Python/CP-SAT timetable solver. It deliberately does **not** reference
this suite's code or schemas — a firewall, not an oversight, meant to keep
the fresh model-first attempt from inheriting this codebase's assumptions.
The one concept imported across the firewall is `ClashDetector`, an
independent verifier sharing no code with the solver — see
[[independent-verification]]. See `CLAUDE.md` in this wiki for further
detail on how the two relate.
