# TimeEduSuite Tools & Archive: Converter Pipeline and Lessons Learned

## Part 1: Converter Tools Overview

### Data Pipeline Architecture

**Formats in the pipeline:**
- **Source:** Microsoft Access `.accdb` database (tables ST1/TT1/LA1)
  - ST1 = student timetable grid (7 periods × 8 blocks, plus P1–P4 out-of-timetable slots)
  - TT1 = teacher timetable grid (same structure)
  - LA1 = activity/free-period master data
- **Intermediate:** Excel `.xlsx` (ST1.xlsx, TT1.xlsx; replicate the accdb column layout)
- **Canonical output:** `timetable.json` (schema v2.1 or v3.0)
  - Student placements, teacher assignments, free periods, subject rosters, venues
  - Consumed by: TimeBuilder (C++), TimeEditor (Qt), TimeView (web client), TimeVerify

**Why conversion is needed:**
- The school's source of truth is the accdb; the JSON is the canonical exchange format for all downstream tools.
- The xlsx intermediate exists for manual data entry and hand-edits; both xlsx and accdb must round-trip to JSON identically.
- Rollover reads the JSON at S0 and writes it at S8, so converter must emit structurally valid, unambiguous lessons.

---

### Converter Scripts (C.5, C.6 sessions)

**1. `timetable_to_json.py`** — **Canonical xlsx→JSON producer**
   - Merges ST1.xlsx + TT1.xlsx into a single schema-v2.1 JSON.
   - Reads student placements (cell text = `SUBJ TEACHER` or activity code) and teacher placements.
   - Parses lessons from `(teacher, subject, grade)` tuples; maps students to lessons via cells.
   - Includes **C.1 lesson-fusion fix:** detects and splits duplicate teacher offerings in the same subject/grade using disjoint-cohort signature.
   - Runs data-quality checks: `dup_placement` (cell occupancy > multiplicity), `lone_student_stray` (minority-size cohort under a teacher/slot).
   - Output validated against schema; logs version + source timestamp.

**2. `accdb_to_json.py`** — **Access→JSON reader (C.5 session)**
   - Reads authoritative school accdb via PowerShell `System.Data.OleDb` (provider: `Microsoft.ACE.OLEDB.16.0`).
   - Dumps ST1/TT1/LA1 to CSV in temp dir, reshapes to xlsx positional layout, reuses `timetable_to_json` internals.
   - Handles OOT (out-of-timetable) P1–P4 slots identically to timetable slots.
   - Command: `python accdb_to_json.py <db.accdb> <out.json>`

**3. `json_to_accdb.py`** — **JSON→Access write-back (C.6 session)**
   - Inverse of C.5; writes edited/constructed JSON back into a fresh accdb copy.
   - **ST1 rewrite:** per student, reconstructs cells A1–H7 + P1–P4 from `student_slots` + `free_periods.students`; renormalizes option codes (e.g., `OM→MU`).
   - **TT1 rewrite:** derives teacher occupancy from `lessons[].teacher × placements`; reconstructs cell text as `{grade} {code} {surname}({venue})`.
   - **LA1:** copy-through unchanged.
   - **Never mutates source:** always writes to separate output; includes round-trip verification (`--verify` flag).
   - Command: `python json_to_accdb.py <in.json> <template.accdb> <out.accdb> [--verify]`

**4. `validate.py`** — **Schema v3.0 validator**
   - Standalone check of JSON structure, field presence, invariants.
   - **Hard invariants** (export-blocking): version match, required fields, referential integrity.
   - **Soft checks** (warnings only): placement multiplicity, orphaned lessons, free-period codes.
   - Exit codes: 0 = clean, 1 = hard violation, 2 = parse error.
   - Command: `python validate.py <timetable.json> [--strict-version]`

**5. `subjectData.py`** — **Subject metadata lookup**
   - Reads school subject configuration (code → grade → multiplicity `M`).
   - Used by all parsers to validate lesson contact time and derive multiplicity from subject code + grade.

---

## Part 2: Archive Docs — Bugs, Decisions, and Lessons

### CONVERTER_BUG.md — The Lesson-Fusion Bug (C.1)

**Status:** RESOLVED (2026-06-02). Implementation shipped in `timetable_to_json.py`.

**The bug:**
When a single teacher teaches **two different classes of the same subject in the same grade**, the converter keyed lessons by `(teacher, subject, grade)` and fused both sections into one lesson with `M = 14` and two disjoint student cohorts, instead of emitting two separate `M = 7` lessons.

**Why destructive:**
- **Unplaceable.** `M = 14` exceeds the 7-period block; Builder's placement can't fit it.
- **Phantom clashes.** Fused cohorts that don't actually co-meet produce false clash detections.
- **Silent failure.** No signal that data was corrupted until a downstream tool fails.
- **Vendored bug.** Converter is shipped in TimeView and TimeVerify, so the phantom lesson appears everywhere.

**Fix shape (implemented):**
1. **Unique lesson key:** `(teacher, subject, grade, instance)` where instance increments on triple repeats.
2. **Disjoint-cohort split rule:** if incoming students share no overlap with existing cohort under the same key, allocate a new instance rather than merge.
3. **Loud self-check:** refuse to emit lessons with `M > 7` or student sets that fracture into disjoint components; count source offerings vs. emitted lessons; mismatch fails the run.
4. **Shared field with Builder:** `instanceIndex` (see LESSON_REFACTOR.md §3.2) disambiguates duplicate offerings.

**Test fixture:** any grade where one teacher teaches two sections of a subject. Converter must emit two `M = 7` lessons, never `M = 14`.

---

### LESSON_REFACTOR.md — Eliminating String Parsing (T.0, Builder)

**Status:** RESOLVED (approved 2026-06-02). Targets TimeBuilder session T.0.

**The problem:**
Lesson and subject names were composites (e.g., `"MA_10_Smith"`, `"MA_10_Smith_2"`, `"MA_10_Smith_b"`). Multiple code sites parsed these strings to recover grade, subject code, and sibling status, coupling logic to naming convention. This:
- Baked brittle conventions (e.g., `endsWith("_b")` for Senior MA extension) into algorithms.
- Blocked shared-core extraction—the repair engine contract forbids subject-label parsing.
- Made renaming risky.

**Solution:**
Add fields to carry semantic meaning directly. Identifiers remain unique strings; data lives in fields.

**New fields added:**
- `Subject.code` — subject code (e.g., "MA"); source of truth, not parsed from name.
- `Subject.grade` — already present; no longer derived from `name.split('_')`.
- `Lesson.subjectCode` — mirrors `Subject.code`; avoids hot-path `subject.code` chain.
- `Lesson.grade` — already present; not reparsed from name.
- `Lesson.instanceIndex` — for duplicate teacher offerings (1 for first, 2 for second, etc.). Replaces `name.split('_')` at parse sites.

**Parse-site changes (audited):**
- `datasets.cpp:157` — stopped parsing `Subject.name` to recover code/grade; read from fields.
- `datasets.cpp:300, 328` — `getLessonsBySubject` / `reservedBySubject` changed to compare `Subject.code` field instead of `startsWith`.
- `studentallocator.cpp:534, 571` — replaced `endsWith("_b")` with `M == 7` filter (option A: no explicit `isExtension` flag).

**Verification gate:**
Post-refactor, these checks must pass: zero `QString::split` on domain identifiers, zero `endsWith("_b")`, zero `startsWith(subjectName + "_")`, and the allocator log byte-identical to pre-refactor baseline.

**Cross-link:** `instanceIndex` is the shared field that both the Builder (T.0) and the converter (C.1) use to disambiguate duplicate offerings.

---

### EJECTOR_PLAN.md — Phase C Repair Engine Design (T.4)

**Status:** PLANNING (design doc, staged build plan T.4a–e).

**Context:**
Phase A (greedy first-fit placement) leaves ~100 "wedged" lessons (cells placed < `M`). Phase C (the ejector) must resolve these by cascading placements. Tractability question: is 100 wedges too many to handle?

**Core insight — Monotone Φ:**
Define **Φ = total unplaced cells = Σ(M − cells.size())** over all lessons.
- Each ejector transaction (one deficit resolution) is atomic: commit → Φ decreases ≥1, rollback → Φ unchanged.
- Φ is bounded below by 0, starts finite ⇒ **termination guaranteed**. Not a 100-dimensional explosion; ~100 independent, individually-bounded steps.

**Architecture (two layers):**

**Primitive `resolveOneDeficit(target, depthLimit, budget, tx) → Outcome`:**
The bounded ejection chain for one deficit. Tries:
1. Free placement (no ejection).
2. Structural clearing (Y-axis move, student rehome/swap, no recursion).
3. Recursive ejection (eject a blocker, re-place it deeper, within `depthLimit`).
Atomic: commit ⇒ cell placed + chain complete, rollback ⇒ no mutation.

**Driver `run()` (Builder-specific):**
Collects all deficits, orders by MRV (most-constrained first), invokes primitive in three progressive layers:
- **Layer 0:** direct placement (depthLimit=0); absorbs easy wedges that have free cells.
- **Layer 1–3:** escalating depth; each pass shrinks survivors before depth increases.

**Locked design decisions (D1–D8):**
- D1: Monotone Φ invariant (one transaction per deficit).
- D2: **Single `isTeacherClash` chokepoint** for PE exemption rule.
- D3: Pin senior PE/LO bands as immovable; route around as hard blockers.
- D4: Transaction = entire chain, never partial.
- D5: Three-layer strategy (Layer 0 direct → progressive-depth ejection → MRV).
- D6: Occupancy rebuilt from `cells` after loop (spec decision §7).
- D7: Per-transaction visited-set; reset per deficit.
- D8: Stuck-tail policy = report-and-stop with diagnostics (v1, no silent auto-escalation).

**Budgets (quantitative, tied to ~100 wedges):**
- Global: `EJECTION_NODE_BUDGET = 50,000` (or ~150,000 if robustness to full 100 is required).
- Per-target: `EJECTION_PER_TARGET_BUDGET = 500`.
- Max depth: `EJECTION_MAX_DEPTH = 3`.
- Arithmetic: 100 × 500 = 50,000; Layer 0 + Phase A improvement are load-bearing to stay under budget.

**Observability (built into T.4a):**
Emit:
- **Resolution histogram:** count wedges resolved at depth 0/1/2/3/failed.
- **Failure taxonomy:** failed wedges grouped by blocker kind + block.
- **Φ trace:** Φ after each layer (must be monotone).
- **Budget usage:** `nodesExpanded` total vs. cap.
- **Spec line:** `[phaseC:done] clashes_in=<a> repaired=<b> residual=<c>`.

Enables data-driven ejector-vs-Phase-A decision: if stuck tail is uniformly TEACHER-blocked in one block, that's a Phase-A load-balance fix, not an ejector-depth fix.

**Staged build (T.4a–e with gates between):**
- T.4a: Scaffolding + Layer 0 + observability. Gate: Layer 0 clears N wedges, zero constraint violations.
- T.4b: Depth-1 structural clearing. Gate: Φ strictly down on commit, flat on rollback.
- T.4c: Recursive ejection + progressive driver. Gate: stuck wedges clear, budgets respected.
- T.4d: Cross-grade teacher clash repair + cube verify. Gate: zero unplaced or documented stuck tail.
- T.4e: Hardening for other drivers (Editor manual-edit, rollover 200-defect set). Gate: primitive unchanged on both.

---

### LESSON_REFACTOR.md & EJECTOR_PLAN.md Convergence

Both docs target the same field: `instanceIndex`. 
- **Builder (T.0, LESSON_REFACTOR):** carries the instance number to avoid re-parsing lesson names.
- **Converter (C.1, CONVERTER_BUG):** uses the same field to distinguish duplicate teacher offerings in JSON.

The connection is load-bearing: Builder reads `instanceIndex` from JSON (T.5 session), and rollover depends on converter output being correct (C.1 ships before E.7). Shared-field design keeps them aligned.

---

### T5D_BUILD_LOG.md — Session 6 State and Tracked Debt

**Status:** Mixed. T.5d (JsonContract + Builder export) clean; Editor integration debt remains.

**T.5d Gate Results (3-part, all PASS):**
1. **`validate.py`** on Builder 2025 export → 0 HARD violations.
2. **Round-trip stable:** `write(read(write(read))) == write(read)` ✓
   - Fix (C3): `instanceIndex` recovery from label suffix (e.g., `AF_RUST_09_2`).
   - Fix (C4): Deterministic QHash seed to avoid student-slot write order variance.
3. **Builder-vs-accdb entity compare:** students/slots identical; teachers 46 vs 55 (9 OOT-only), subjects 265 vs 322 (label granularity). All known deltas.

**Pre-existing debt (not T.5d-introduced):**
- Editor broke on T.5 constant renaming (`kNoOfBlocks` → `NO_OF_BLOCKS`, etc.). Fixed in-session per Eugene's approval (option b: update Editor to match Builder constants).
- 2026 data missing TT12026; feasibility unconfirmed.

**Measured state (2025, accdb-derived):**
- Pre-ejector: `residual=40 wedges`, `cells=1180/1309` (90% placed).
- Post-ejector: same 40 remaining (goal <20). Gap is placement-algo quality (Phase A greedy → weak), not data.

---

## Part 3: Key Decisions & Lessons

### Schema Stability vs. Extensibility
- **v2.1 locked:** output must satisfy all invariants in SCHEMA.md §5 (no minor-version bumps without approval).
- **v3.0 prepared:** JSON→Access write-back (C.6) uses v3.0; reader path (`readTimetableJson`) exists but full round-trip harness not yet run.

### Data Quality is Visible
- DQ checks (`dup_placement`, `lone_student_stray`) run on both xlsx and accdb paths; surface warnings for human review, not auto-fix.
- 2025 baseline: 17 dup + 17 stray = known data-entry anomalies; genuine hard errors still stopped the run (e.g., unmatched teachers).

### Surname Collisions Require Multi-Signal Tie-Break
- Resolution order: **code → grade → TT1 slot-overlap → venue → display_name**.
- Slot-overlap (the decisive signal) comes from TT1 (source-of-truth teacher grid); venue/display_name are discriminators.
- Affected both C.5 (read) and C.6 (write) paths; ST1 cells carry only `CODE SURNAME` (no initial/venue), so TT1 is needed.

### Vendored Converter = Single Source of Truth
- `timetable_to_json.py` is vendored into TimeView, TimeEditor, TimeVerify at release.
- Edit here, not in copies. Bug fixes (e.g., C.1 lesson-fusion) must ship everywhere simultaneously.
- OOT (out-of-timetable) handling is identical across all consumers by reusing the same code.

### Round-Trip Gateway, Not Byte-Identical
- `accdb → JSON → accdb' → JSON'` gate checks JSON equality, not cell-text byte identity.
- Normalization is acceptable: option codes renormalize (OM→MU), surnames re-parse, teacher nulls handled (`UNKNOWN`).
- **Baseline must be fresh accdb read**, not hand-edited JSON (2025 timetable.json was edited by `fix_clashes_2025.py`; not a valid baseline).

### Write-Back is ST1-Only (TT1 Derived)
- Schema v2.1 doesn't store per-slot teacher placements; TT1 cells carry class# + code, not kept.
- TT1 editing isn't directly reconstructable; TT1 is left as-is and derived from JSON on re-read.
- v2.2 bump only if team-teaching (>1 teacher per label) or school-token preservation becomes hard requirement.

---

## Part 4: Open Work & Blockers

- **E.7 (Editor end-to-end rollover):** depends on C.1 (converter fusion fix) + T.5d (JSON read exist).
- **2026 data:** `ST12026.xlsx` ready, but no `TT12026` (stale per HANDOFF); feasibility unconfirmed.
- **Builder placement quality:** Phase-A best-fit / backtrack improvements planned (Phase B König matching, Phase C ejector) to close wedge gap from 40 → <20.
- **Ejector driver-agnostic hardening (T.4e):** primitive works on 1-defect (manual edit) and 200-defect (rollover) sets identically; ready for wrapping as `EjectorRepairEngine : IRepairEngine` at T.5.

---

## Sources

- `E:\TimeEduSuite\tools\converter\CLAUDE.md` — converter project guide, C.1 reference.
- `E:\TimeEduSuite\tools\converter\HANDOFF.md` — C.5/C.6 session summary, data layout, gotchas.
- `E:\TimeEduSuite\tools\converter\accdb_to_json.py` — Access→JSON reader.
- `E:\TimeEduSuite\tools\converter\json_to_accdb.py` — JSON→Access write-back.
- `E:\TimeEduSuite\tools\converter\timetable_to_json.py` — xlsx→JSON canonical producer.
- `E:\TimeEduSuite\tools\converter\validate.py` — JSON schema validator.
- `E:\TimeEduSuite\tools\converter\subjectData.py` — subject metadata.
- `E:\TimeEduSuite\docs\archive\CONVERTER_BUG.md` — C.1 lesson-fusion bug & fix.
- `E:\TimeEduSuite\docs\archive\LESSON_REFACTOR.md` — T.0 field-driven refactor (Builder).
- `E:\TimeEduSuite\docs\archive\EJECTOR_PLAN.md` — T.4 Phase C repair engine design.
- `E:\TimeEduSuite\docs\archive\T5D_BUILD_LOG.md` — session 6 build log & state.
