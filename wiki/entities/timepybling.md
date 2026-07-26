---
tags: [entity, timeedusuite, legacy-attempt, archived]
created: 2026-07-26
updated: 2026-07-26
sources: [raw/sources/timeedusuite-archived.md]
---

# TimePyBling

Archived attempt (`E:\TimeEduSuite\Archived\TimePyBling`), part of
[[timeedusuite-suite]] history. South African school exam scheduler: reads a
student timetable, detects double-bookings, auto-generates a clash-free
cross-grade exam schedule.

## Approach

Python 3.13 + Tkinter, layered (pure core → reader → app → UI). 3-phase
constructive algorithm (pinned blocks → red-paper 5-day-spaced AM slots →
yellow/green fill) plus 200-pass hill-climb optimization. Cost function
penalizes per-student overlap windows. Graph colouring (DSatur +
backtracking) for clash resolution.

## Why abandoned (2026-06-02)

**Not an algorithm failure** — the scheduling approach worked.
Python/Tkinter performance was insufficient for large school datasets.
Deferred pending a C++ port (planned as "TimeExam"), which awaits the shared
[[timeedusuite-core-library]]. Preserved in place as the design template for
that port.

## Known bugs, deferred (not re-fixed in Python)

JSON load duplicates papers instead of replacing; penalty-breakdown UI dead;
5-day red-spacing enforcement unverified; navigate-to-cell unverified.

See [[python-prototype-to-cpp-pattern]] for the general shape this follows.

Full detail: `raw/sources/timeedusuite-archived.md`.
