# TimeEduSuite Archived Attempts

Abandoned/superseded school timetabling tool branches. Document captures lessons and blockers to avoid re-discovering dead ends.

## TimePyBling

**Purpose:**
South African school timetable analysis and exam scheduling. Reads a student timetable (Excel), detects double-bookings, and auto-generates a clash-free, cross-grade exam schedule with dated slot assignments.

**Approach:**
- Stack: Python 3.13, Tkinter GUI, pandas/openpyxl for file I/O
- Layered architecture: pure core (domain logic) → reader (scheduling) → app (orchestration) → UI (tkinter)
- 3-phase constructive algorithm: Phase 0 (pinned study blocks) → Phase 1 (red papers, 5-day spacing, AM only) → Phase 2/3 (yellow/green fill) → hill-climb optimization (200 passes, paper swaps)
- Exam papers labeled by difficulty (red/yellow/green) with linkage support (e.g., Music P1+P2 same-day)
- Cost function: per-student overlap windows (2/3/4-day convolution weights 3/2/1)
- Graph coloring (DSatur + backtracking) for exam clash resolution

**Why Abandoned (2026-06-02):**
Python/Tkinter performance insufficient for large school datasets. Becoming a source reference for C++ port (TimeExam). Preserved in place as design template—core algorithms (conflict_matrix.py, exam_clash.py, exam_tree.py, exam_scheduler.py, cost_function.py, state_repo.json schema) will be ported to C++ when TimeExam builds.

**Known Bugs Deferred to C++ Port:**
1. JSON load duplicates papers (adds on top instead of replace)
2. Penalty breakdown UI dead (penalty_log populated but no display)
3. 5-day red spacing enforcement unverified
4. Navigate-to-cell (scroll to subject matrix cell) unverified

**Dates:**
- Last session: 2026-04-02 (Session 9)
- Archived: 2026-06-02
- Status: **Deferred** per suite strategy; C++ successor (TimeExam) awaits shared core library ship date

---

## TimeVerify

**Purpose:**
School timetable verification and validation tool. Reads student/teacher timetables (Excel), detects clashes (double-bookings, out-of-subject teacher assignments), allows edit-with-undo, and exports validated schedules to JSON for downstream tools (TimeView, TimeExam).

**Approach:**
- Stack: Python 3.13, Tkinter GUI, pandas/openpyxl, no database
- Layered architecture: pure verify (clash/integrity checking) → app (orchestration) → file_io (Excel/JSON) → UI (tkinter)
- Two verification passes: student double-bookings, teacher qualification (requires teacher-subject pool file)
- Interactive clash resolution: students can change subjects with undo/redo support via ChangeStudentSubject edit class
- Export gate: clashes block export; warnings allow export with confirmation
- State: edit stack pattern for undo/redo, dirty flag tracking

**Why Abandoned (2026-06-02):**
Fully superseded by C++/Qt rewrite at TimeEditor (parallel sessions E.3 + E.4 ported clash_checker.py, integrity_checker.py, edit stack pattern, and ChangeStudentSubject/MovePlacement edits). Preserved as port reference and verification oracle (CLI parity test for C++ version).

**Scheduled Moves (Session C.0):**
- `file_io/timetable_to_json.py` → `..\tools\converter\`
- `file_io/subjectData.py` → `..\tools\converter\`

**Dates:**
- Session 6: Student subject-change UI completed
- Session 7 Goal: 2026-04-30 (JSON writeback for enrolment changes)
- Archived: 2026-06-02
- Status: **Superseded** by TimeEditor C++/Qt version

---

## Key Lessons

1. **Python Tkinter insufficient for production timetable tools** — both attempted Python GUI desktop apps; both deferred to C++ (TimeExam, TimeEditor)
2. **Layered architecture essential** — both projects enforced strict core/reader/app/ui separation; this pattern carried to C++ ports
3. **Exam scheduling is hard; two approaches attempted:**
   - TimePyBling: 3-phase constructive + hill-climb (succeeded algorithmically; perf was the blocker)
   - TimeExam (planned): C++ rewrite of same algorithm for speed
4. **Verification-first workflow** — TimeVerify's clash-detection + edit-with-undo pattern is being ported to TimeEditor, suggesting it was the right UX model

---

## Sources

- `E:\TimeEduSuite\Archived\TimePyBling\ARCHIVED.md`
- `E:\TimeEduSuite\Archived\TimePyBling\HANDOFF.md`
- `E:\TimeEduSuite\Archived\TimePyBling\README.md`
- `E:\TimeEduSuite\Archived\TimePyBling\CLAUDE.md`
- `E:\TimeEduSuite\Archived\TimeVerify\ARCHIVED.md`
- `E:\TimeEduSuite\Archived\TimeVerify\Docs\HANDOFF.md`
- `E:\TimeEduSuite\Archived\TimeVerify\CLAUDE.md`
