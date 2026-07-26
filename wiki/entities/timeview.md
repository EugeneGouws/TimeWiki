---
tags: [entity, timeedusuite, component, live, stable]
created: 2026-07-26
updated: 2026-07-26
sources: [raw/sources/timeedusuite-timeview.md]
---

# TimeView

**Status: stable, live in production.** Not a legacy attempt — the one
component of [[timeedusuite-suite]] currently deployed and used by real
schools. v1 shipped Session 11, pilot 2026-07 at Crawford International
College La Lucia.

## What it is

Read-only, browser-based personal timetable viewer. React 18 + Vite, no
backend, no login (single upload key gates the only protected action).
Renders the 8×7 grid matching the school's printed timetable, with live
search (student/teacher/subject) and a 3-way comparison overlay.

## Key technical decision: no persistent storage

Timetable JSON lives in browser memory only — File System Access API handle
(Chrome/Edge) or plain file input with no persistence (Firefox/Safari),
re-read from disk every load. Driven by **POPIA compliance** (South African
data protection law) — an earlier localStorage-based version (v1–10) was
replaced for this reason. This is a hard rule, not a preference: "never
cache timetable.json blob in persistent storage."

## Lessons learned (from its own MISTAKES.md — worth generalizing)

1. Never copy merge tables/assumptions from sibling projects without
   verifying against this school's actual data.
2. Read the actual data file, not just a bug description — descriptions can
   describe a *previous* bug.
3. Schema is reality, spec is intent — build fields from the schema, not the
   spec doc.
4. Walk the full flow end-to-end before shipping, especially after changing
   something upstream (auth, routing).

## Workflow rule (worth noting as a pattern)

Plan-first: architect (Eugene) reviews plan before code, waits for approval,
owns git/deploy. Builder (Claude Code) never pushes to git, never modifies
CLAUDE.md.

## v2 backlog highlights

Remove dead xlsx-era files, cap long activity-cell lists, FastAPI backend
for protected upload, venue search, PDF export.

Full detail: `raw/sources/timeedusuite-timeview.md`.
