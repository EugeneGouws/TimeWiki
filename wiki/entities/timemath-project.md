---
tags: [entity, project, timemath]
created: 2026-07-26
updated: 2026-07-26
sources: [raw/sources/timemath/CLAUDE_root.md, raw/sources/timemath/docs/PRODUCT.md, raw/sources/timemath/docs/PLAN.md, raw/sources/timemath/docs/HANDOFF.md, raw/sources/timemath/docs/TOKENS.md, raw/sources/timemath/docs/BRAINSTORM.md, raw/sources/timemath/solver-docs/CLAUDE.md, raw/sources/timemath/solver-docs/PLAN.md, raw/sources/timemath/solver-docs/CONSOLIDATION.md]
---

# TimeMath (repo `E:\timemath`)

6th attempt at a school-timetable solver. Prior five failed by coding before
the model was understood — working rule now: nothing goes into code until
checked against real data and small enough to verify by hand. Model-first,
Python + OR-Tools CP-SAT.

Distinct from **[[timeedusuite-suite]]** (C++/Qt production track,
TimeEditor/ TimeVerify/TimeView/TimePyBling) — timemath deliberately shares
no code or schema with it, only borrows the *concept* of an independent
verifier (`ClashDetector`).

## Status (as of 2026-07-26)

Phase 0 (normalise) and Phase 1 (bounds) done, validated in Excel
(`Phase01_2026.xlsx`) across Gr10–12, both years. Phase 2 (precolour) is the
active coding front, scoped to Grade 12 2026 first. Phases 3–5 are
**analytically explored only** (see [[phase3-column-student-assignment]],
[[phase4-teacher-assignment]], [[repair-model]], [[warm-start-rejected]]) —
nothing beyond Phase 2 exists as code.

**Doc-tree status (re-checked 2026-07-26, later same day):** the repo root
`CLAUDE.md` now *says* the `solver-docs/` staging draft has been merged in
and `solver-docs/` deleted — but on disk `solver-docs/` still exists in full
and `docs/PLAN.md` is unchanged. The prose claim and the filesystem
disagree; treat root `CLAUDE.md`'s "done" claim as unverified. See
[[solver-docs-consolidation]] for the full trace — not resolved here, by
design (that decision belongs in the timemath repo itself).

## The pipeline

See [[timetable-solver-pipeline]] for the full Phase 0–5 breakdown. Short
version: Phase 0 parses + validates inputs and derives baskets; Phase 1
computes lower bounds and fails fast with a named culprit; Phase 2 pins
symmetry breaks and shrinks domains without placing anything; Phase 3
assigns students to sections to columns (CP-SAT); Phase 4 assigns teachers
to sections, minimising distinct teachers; Phase 5 expands placed sections
into the actual grid and reinstates quarantined exception students.

## Key established facts

* Timetable shape: 8 columns (A–H) × 7 periods = 56 cells, plus 4
  out-of-timetable fields (P1–P4).
* `MA_S` = 10 (not 7 — the JSON's stale `m` field lies), `ML` = 7.
  Compulsory band = EN (1 column) + MA/PE/LO packed into 2 more = 3 columns
  fixed, 5 left for choice. Zero slack — see [[universal-block-and-band]].
* Binding constraint is **basket-level SDR** (System of Distinct
  Representatives via Hall's theorem), not pairwise subject conflict — see
  [[basket-sdr]].
* Section-level, not subject-level, is the correct unit for column
  assignment: a multi-section subject can legally span multiple columns.
  Subject-level all-different is only sound for floor=1 (singleton-section)
  subjects.
* Teacher qualification pools cap *simultaneous* sections per column, not
  total sections a teacher can run across the timetable.
* Regression fixtures, subject-grade existence table, code-drift table
  (`CAT`→`CA`, `OD`→`ODR`, `RDI`→`RD`): full detail in the source CLAUDE.md
  files, not duplicated here.
* Five Gr10 rows in the 2026 workbook share a duplicate student ID — dedupe
  before trusting Gr10 fixtures built off raw row counts.
* `GOVENDER` covers 3 distinct teachers, `WYLIE` covers 2, in the teacher
  data — rebuilt `TT<year>.xlsx` splits them (`GOVENDER_K/L`, `WYLIE_C/Z`);
  legacy files carry phantom double-bookings until disambiguated this way.
* The 2026 teacher table in the original JSON export is a stale copy of
  2025's — never source teacher identity from it, reconstruct from section
  keys instead.
* Warm-starting the solver from a prior year is **rejected**: year-to-year
  cell-placement Jaccard similarity is only ~0.19 despite stable structure,
  and it would bias the solver toward defending last year's teacher count —
  the exact number Phase 4 is trying to minimise. See
  [[warm-start-rejected]].
* Now confirmed in root `CLAUDE.md` (not just the staging draft): the
  TimeEduSuite firewall (see [[timeedusuite-suite]]) is adopted as project
  policy, not just a staging-draft proposal.

## Working mode

Guided coding: user (Eugene) writes all project code, Claude coaches — never
writes large files, only diffs/snippets. Token economy is a standing concern
— see [[timemath-token-economy]]. `roster.py` is the module in progress.
