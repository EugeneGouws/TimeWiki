---
tags: [concept, timeedusuite, pattern, process]
created: 2026-07-26
updated: 2026-07-26
sources: [raw/sources/timeedusuite-archived.md, raw/sources/rollover.md, raw/sources/timeedusuite-timebuilder.md, raw/sources/timeedusuite-timeeditor.md]
---

# Python prototype → C++ rewrite pattern

A shape that recurs three times across [[timeedusuite-suite]]'s history:
prototype a tool in Python (fast to write, good for proving an algorithm or
UX), then rewrite it in C++/Qt once it needs to be fast, integrated, or
production-grade.

## The three instances

1. **[[timepybling]] → TimeExam** (planned). Reason for rewrite: pure
   performance — Python/Tkinter too slow for large school datasets. The
   algorithm itself (3-phase constructive + hill-climb) was judged sound and
   is the design template for the port.
2. **[[timeverify]] → [[timeeditor]]**. Reason for rewrite: integration, not
   performance — needed the shared [[ejector-repair-engine]] and a proper
   desktop UX. The verification-first UX pattern (detect clashes, fix
   interactively with undo) was ported essentially unchanged; only the
   implementation language changed.
3. **[[rollover-cpp-prototype]] → [[timeeditor]]'s rollover pipeline**.
   Reason for rewrite: completeness — the prototype detected clashes but
   couldn't repair them and had no JSON export. Both are C++, so this isn't
   strictly cross-language, but it's the same "standalone proof of concept,
   then absorbed into the shared-core suite" shape.

## What carries over vs. what doesn't

**Carries over:** layered architecture (pure-core / io / app / UI separation
was enforced in every Python prototype and inherited by the C++ ports), the
core algorithm or UX pattern, domain rules (e.g. grade-advancement quirks in
[[rollover-year-advancement]]).

**Gets rebuilt:** the actual implementation, string-based domain-object
identity (see `instanceIndex` in [[json-timetable-schema]] — the C++ side
deliberately moved away from name-parsing that the Python prototypes used).

## Why this is worth remembering

Both Python attempts were **archived as reference material in place**, not
deleted — `ARCHIVED.md` files explicitly frame them as port templates, not
failures. The lesson generalizes: a prototype that proves a design but hits
a non-algorithmic wall (speed, integration) is worth keeping as a spec, not
just as a postmortem.
