# TimeExam — port notes from TimePyBling

What survives from the archived Python attempt, what is deliberately
dropped, and what is inherited debt.

## Source availability — read this first

**TimeWiki holds almost nothing about TimePyBling's internals.** The total
recorded material is ~60 lines:

- `wiki/entities/timepybling.md` — ~25 lines
- `raw/sources/timeedusuite-archived.md:5–30` — 25 lines, itself a
  distillation of four upstream docs that were never copied into TimeWiki
- `wiki/concepts/python-prototype-to-cpp-pattern.md:17–20` — 4 lines
- Scattered one-line forward references in four other files

The upstream detail lives at `E:\TimeEduSuite\Archived\TimePyBling\` in
`ARCHIVED.md`, `HANDOFF.md`, `README.md` and `CLAUDE.md`. **Anyone
implementing TimeExam should read those first** — everything below is
reconstructed from the distillation, not from the source.

In practice this matters less than it sounds, because the `DESIGN.md`
scoring model is substantially new. TimePyBling is a reference for what the
problem *feels* like, not a specification to port.

## Carried over

**The consecutive-day convolution cost.** TimePyBling's cost function used
"per-student overlap windows (2/3/4-day convolution weights 3/2/1)". This is
the single most valuable thing it proved, and `DESIGN.md` §4.1 keeps it —
extended to a 5-day window and normalised per student. The 3/2/1 weights
seed the new `λ_w`.

**Paper linkage.** "Linkage support (e.g., Music P1+P2 same-day)" becomes
link groups (`DESIGN.md` §2.4), generalised to also carry pinned blocks and
cross-grade splits.

**Layered architecture.** TimePyBling enforced pure core → reader → app →
UI, and the archived notes call this out as a key lesson that "carried to
C++ ports". `REPO-SKELETON.md` follows it.

**Global local search.** The 200-pass hill-climb over paper swaps was
horizon-wide, not week-sequential. Keep that property (`DESIGN.md` §5).

**Red/yellow/green as a concept, not a mechanism.** The old difficulty
labels are subsumed by `SF` — a 1–5 integer is the same idea with more
resolution and no phase-ordering baggage.

## Deliberately dropped

**The single-score model.** `students × SF` conflates student stress,
marking effort and invigilation demand. See `DESIGN.md` §0.1.

**Phase gating.** TimePyBling's "Phase 0 pinned → Phase 1 red papers with
5-day spacing, AM only → Phase 2/3 yellow/green fill" hard-wired the
front-loading preference into the construction order. `DESIGN.md` makes it
a weighted cost term (§4.4) instead, so the optimiser can trade it off
rather than being bound by it.

**AM-only restriction on heavy papers.** Replaced by a PM stress multiplier
(§4.3) — same intent, expressed as a cost.

**The "5-day spacing" hard rule.** Expressed via the window penalties
instead. Note this rule's enforcement was *never verified* in the original
(see inherited debt below), so there is nothing proven to preserve.

## Inherited debt

Four bugs were explicitly deferred from TimePyBling to its successor rather
than fixed in Python. Three are irrelevant to a rewrite; one is a genuine
warning.

| # | Bug | Relevance to TimeExam |
|---|---|---|
| 1 | JSON load duplicates papers (adds instead of replacing) | Load semantics — make load *replace* state explicitly, and test it |
| 2 | Penalty-breakdown UI dead (`penalty_log` populated, never displayed) | **Build the breakdown view.** With six weighted terms (§4.1–4.5) an unexplainable score is unusable — you cannot tune weights you cannot see |
| 3 | 5-day red spacing enforcement **unverified** | The old tool may never have honoured its own core spacing rule. Do not assume any TimePyBling output was correct; treat it as a UX reference, not a correctness oracle |
| 4 | Navigate-to-cell unverified | UI only |

Bug 3 is the important one: it means TimePyBling **cannot serve as a
verification oracle** for TimeExam, unlike TimeVerify which was explicitly
preserved as a "CLI parity test" for TimeEditor. TimeExam needs its own
independent checker.

## Modules named in the archive

The six files listed as "will be ported to C++ when TimeExam builds". They
are **named and never described** anywhere in TimeWiki — no
responsibilities, no call graph, no schema. Listed here only so the names
are recognisable when the upstream repo is opened:

`conflict_matrix.py` · `exam_clash.py` · `exam_tree.py` ·
`exam_scheduler.py` · `cost_function.py` · `state_repo.json`

`exam_tree.py` is the intriguing one — a tree structure suggests the
backtracking search that accompanied the documented "DSatur + backtracking"
graph colouring. `DESIGN.md` does not use graph colouring (the problem is
load-balancing, not clash-minimisation — clashes are a hard constraint
handled by cohort disjointness), so this may not carry over at all.

## Why it was abandoned — and why that matters here

Archived 2026-06-02. **Not an algorithm failure** — the notes are explicit
that the scheduling approach worked and that Python/Tkinter performance was
the blocker on large school datasets.

No benchmark, dataset size, or profile was recorded, so *where* the time
went is unknown — and no school-size figures exist anywhere in TimeWiki to
reconstruct it from. The most likely candidate, and the thing `DESIGN.md`
§5 guards against hardest, is full cost re-evaluation per candidate move:
an un-incremented objective costs `O(students × days)` per move, and the
documented 200 hill-climbing passes multiply that by every move considered.
Bitset cohorts plus delta evaluation address exactly this.

**This is a hypothesis, not a finding.** Measure before optimising further;
if the new build hits the same wall, the assumption was wrong and the real
profile needs finding.

