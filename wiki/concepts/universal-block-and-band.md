---
tags: [concept, timemath]
created: 2026-07-26
updated: 2026-07-26
sources: [raw/sources/timemath/CLAUDE_root.md]
---

# Universal block and the MA/PE/LO band

[[timemath-project]]'s compulsory block occupies exactly 3 of the 8 columns,
with zero slack:

* `EN` (M=7) fills one column alone.
* `MA_S`(10) + `PE`(2) + `LO`(2) = 14 cells = a 2-column **band**, packed as
  one composite region — not two separately-placed columns. Verified on 2026
  Gr11: an MA student can read `A = MA×6 + PE×1`, `G = MA×4 + LO×2 + PE×1`.
  `PE` and `LO` are not confined to one column of the band.
* `ML` (7) runs parallel inside the band, filling one of its two columns; ML
  students carry 3 slack cells.
* Band width = `ceil(ΣM_band / periods_per_column)` — a formula, not a
  hardcoded fork, so it stays a parameter across years.

5 columns remain for choice subjects. Max basket size is 5, `EN` is
universal so needs a column disjoint from all choice columns, band needs 2:
5 + 1 + 2 = 8 exactly. This is why `EN` cannot split and the band cannot
spill — forced by the column budget, not by convention. Only *which* columns
get used is free (pure symmetry), which is why pinning this placement in
Phase 2 is sound (see [[timetable-solver-pipeline]]).

`MA_S` = 10 confirmed in both 2025 and 2026 — the JSON's `m` field is stale
and reads 7; never trust it (see [[timemath-project]] known-traps).
