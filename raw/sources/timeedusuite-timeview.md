# TimeView — Timetable Viewer for Schools

**STATUS: stable, live in production** (shipped v1, Session 11; pilot 2026-07)

---

## What TimeView Is

TimeView is a **read-only, browser-based personal timetable viewer** for South African schools. It serves one timetable JSON per deployment; every person at the school (student, teacher) can access it without login and view their personal schedule grid. Timetable upload (the only protected action) is gated by a single upload key.

**Problem solved:** Teachers and students need to see their personal timetable (which subjects/periods, with whom, where). TimeView renders this as an 8×7 grid (8 days, 7 periods) matching the school's printed timetable exactly, with colour-coded lesson cells and a reference times table at the bottom.

**Users:** Students, teachers, staff. Primary deployment: Crawford International College La Lucia (South Africa).

**Deployment:** Cloudflare static host. One instance per school.

---

## Architecture & Tech Stack

**Frontend:** React 18 + Vite (single-page app, no router)  
**Styling:** Plain CSS (no Tailwind, no UI library; Crawford branding scoped to REG/BREAK columns)  
**Data:** Single `timetable.json` file, uploaded by administrator (VP/timetable keeper)  
**Storage:** 
- **v1–10:** localStorage blob (`timeview.timetable.v3`) — replaced due to POPIA concerns
- **v10+:** File System Access API handle in IndexedDB (Chrome/Edge only); content never persisted, re-read from disk every load. Firefox/Safari fall back to plain file input with no persistence.
- **Key point:** Data stays in browser memory only; no cached copy on disk (POPIA compliance for South African schools)

**Backend:** None in production v1. (FastAPI prototype in spec; deferred to v2.)

**Data source:** `timetable.json` produced by `st1_to_json.py` (TimeVerify export pipeline). Schema v2.1 required; mismatch warned on load.

---

## Key Features

### Grid & Layout
- **8×7 grid** (8 day-rows, 7 lesson periods per day)
- **Decorative columns:** REG/BREAK between lessons, colour-coded in Crawford colours (#F9CDAA orange, #8ACA6C green)
- **Header** (two rows): date + school name + meta (student shows grade/reg_class; teacher shows venue)
- **Reference times table** (bottom): start/end times per day type (Normal / Assembly / Test / Long Reg)

### Search & Entity Views
- **Live search bars:** Student (by ID/name), Teacher (by code/name), Subject (by code/name)
- **Entity replacement:** Clicking a search result replaces the school-wide grid with that person's personal timetable
- **Comparison overlay (Session 11):** Compare up to 3 entities side-by-side on one grid; colour-coded per-source; green highlight shows shared-free periods

### Cell Content
**Student view:** Subject name + teacher name (prefix: MR/MS/DR)  
**Teacher view:** Grade group (e.g. "8F (D5)") + venue  
**Free periods:** Activity code in italics/dimmed (STUDY, LIB, LIBRARY, BATTING, MEETING)  
**Click drill-down:** Cell → subject list → student roster → individual student timetable (cascade popout)

---

## Technology Decisions

1. **Client-only, no backend:** Simplifies deployment and avoids server state. Data validation happens on upload (JSON schema v2.1 check).
2. **File System Access API over localStorage:** POPIA compliance—timetable blob must not persist to disk. Handles degrade to plain file input on unsupported browsers.
3. **Plain CSS over UI libs:** Ensures tight control over grid layout and Crawford visual fidelity.
4. **Vite over Create React App:** Fast HMR during development; lightweight prod build.

---

## Lessons Learned (from MISTAKES.md)

1. **Never copy merge tables from sibling projects** (Session 1, 2026-04-21)  
   Applied TimePyBling's `SUBJECT_MERGES` without verifying. OD and DR are distinct subjects in this school. Always validate assumptions against the actual data file.

2. **Read the actual file, not just the bug description** (Session 1, 2026-04-21)  
   Bug report said "BL/BM xlsx columns," but real headers were P1/P2. The description matched a *previous* bug, not current data.

3. **Schema is reality; spec is intent** (Session 5, 2026-04-25)  
   Built header fields from the spec that weren't in the schema (students[id].reg, students[id].house). Always read the schema first before wiring fields.

4. **End-to-end walk-through before shipping** (Session 9, 2026-05-08)  
   `handleLoginSuccess` fetched a hardcoded wrong path (`/data/timetable.json`, should be `/timetable.json`). After changing auth flow, re-trace the full flow (login → file browser) to catch typos and mismatches.

---

## Current State (v1, as of 2026-07-24)

**Session 11 merged to main** (commit `c9ad950`, fast-forward; branch deleted; pending Cloudflare push).

**Implemented:**
- Upload: JSON only (file browser, password gated)
- Schema validation: v2.1 consumed; mismatch warned
- Grid: 8×7 A–H × 1–7; REG/BRK decorative columns
- Search: Student/Teacher/Subject live dropdowns; activities (LIB/STUDY/BATTING/MEETING) merged into Subject bar
- Comparison overlay: up to 3 entities, colour-coded, green shared-free highlight
- Grid height: flexbox layout fills available space, sits flush above times grid
- PopoutIn cascade: cell click → subject list → roster → individual timetable (two-panel, auto-height)
- School name: "Crawford International College La Lucia" (hardcoded; schema carries none yet)

**Known limitations:**
- File System Access API: Chrome/Edge only (persistent handles). Firefox/Safari: plain file input, no persistence.
- Activity cells >3 people: shows full list (deferred capping to "(N more)")
- Study/Free cells: students and teachers shown separately (should merge teachers on top)
- ASSEMBLY row in TimesGrid: 13 cells vs 12 in other rows (ragged column alignment)
- Unused legacy files still present: `subjectsData.js`, `subjectNames.js`, `xlsxToTimetable.js`, `tools/` folder, xlsx code path in UploadButton

---

## v2 Backlog (Planned)

**High priority:**
- Remove dead files: `subjectsData.js`, `subjectNames.js`, `xlsxToTimetable.js`, `tools/`, xlsx path in UploadButton
- Merge teacher + student lists in Study/Free cells (teachers on top, keep cell size)
- Cap activity lists >3 to block count only (e.g. "(12 more)")
- MEETING/LIBRARY/BATTING: show teacher name list; cap at 2 + "(N more)"

**Medium priority:**
- FastAPI backend: protected upload endpoint + serve timetable.json
- Venue search bar (requires venue data in schema)
- Export view as PDF
- Comparison overlay follow-ups: popout/header act on overlaid sources (not just primary); activity-type sources in overlay; second-FILE compare

**Low priority / future:**
- Mobile PWA: landscape-lock only viable as installed app (fullscreen required); defer until demand
- Multi-school / hosted mode with auth layer
- ASSEMBLY row column alignment (cosmetic)

---

## Roles & Workflow

- **Eugene** (architect): Reviews plans before code; owns git; manages Cloudflare deployment
- **Gremlin** (Claude Code): Builder; plan-first workflow (present approach, wait approval, implement)

**Hard rules:**
1. Plan before code; wait for approval
2. Never modify CLAUDE.md (Eugene owns it)
3. Never push to git (Eugene handles it)
4. Never cache timetable.json blob in persistent storage (POPIA)
5. Schema version must be checked on load; warn on mismatch
6. `tools/` and `src/` independent (no runtime coupling)

---

## Sources

- E:\TimeEduSuite\TimeView\CLAUDE.md
- E:\TimeEduSuite\TimeView\Docs\TIMEVIEW_SPEC.md (v1.0 founding spec)
- E:\TimeEduSuite\TimeView\Docs\HANDOFF.md (Sessions 9–11, v1 final state)
- E:\TimeEduSuite\TimeView\Docs\MISTAKES.md (4 key lessons)
- E:\TimeEduSuite\TimeView\Docs\PLAN.md (Sessions 0–11 + v2 backlog)
