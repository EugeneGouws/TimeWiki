---
tags: [entity, timemath, meta, contradiction]
created: 2026-07-26
updated: 2026-07-26
sources: [raw/sources/timemath/solver-docs/CONSOLIDATION.md, raw/sources/timemath/CLAUDE_root_2026-07-26b.md]
---

# solver-docs consolidation (contradiction found, not resolved)

**Update (later same-day check, 2026-07-26):** `E:\timemath\CLAUDE.md` now
contains a "Relationship to other projects" section and corrections 5–8 (dup
Gr10 IDs, GOVENDER/WYLIE surname splits, stale 2026 teacher table,
warm-start rejection) — i.e. the staging draft's content *has* been adopted
into the root file's prose. But its own next-steps list marks "Consolidate
`solver-docs/` into `docs/`" as **done, with `solver-docs/` deleted** — and
that is false as of this check: `solver-docs/` still exists on disk in full,
and `docs/explorations/` / `docs/archive/` do not exist. `docs/PLAN.md` is
unchanged from the pre-update snapshot, so the "PLAN.md merged" half of the
claim is also not (yet) true for that file. **This page now tracks a live
contradiction inside the project's own source of truth, not just an unmerged
draft** — do not treat the "done" claim as fact until re-verified against
the filesystem. See `raw/sources/timemath/CLAUDE_root_2026-07-26b.md` for
the full delta.

## Original finding (first ingest, earlier 2026-07-26)

`E:\timemath\solver-docs\` is a staging folder drafted 2026-07-26 by a
claude.ai chat session ("chat-Claude"), containing an updated CLAUDE.md,
PLAN.md, and `docs/explorations/*` meant to be merged into the repo root —
**not yet merged as of this ingest.** The repo root's own `CLAUDE.md` /
`docs/PLAN.md` are a separately-evolved copy; the two have diverged (repo
root is the one Claude Code has been working against — it has different/
additional content, e.g. `TT<year>.xlsx` conventions, ED_12 at-cap
assertion, exception reinstatement rule not identically phrased).

## What the staging draft claims changed vs the older docs

* CLAUDE.md: current-state section, corrections 4–9 (Phase 3 model fix, 2025
  silent-zeros bug, dup Gr10 IDs, GOVENDER/WYLIE surname splits, stale 2026
  teacher table, warm-start rejection — see [[warm-start-rejected]]), SDR
  soundness boundary ([[basket-sdr]]), band `ceil(ΣM/7)` formula
  ([[universal-block-and-band]]), ED_12 at-cap assertion, `TT<year>.xlsx`
  conventions, exception reinstatement rule, TimeEduSuite firewall
  paragraph, VERIFY flag on the Gr12 fixture row.
* PLAN.md: §5 rewritten to student-to-section formulation (see
  [[phase3-column-student-assignment]]); §6 reframed to
  feasible-or-min-cost-repair (see [[repair-model]]); §8 non-zero-scope
  guards + at-cap assertion; teacher file renamed `TT<year>.xlsx`.
* BRAINSTORM.md moved to an archive folder with a superseded banner (its "no
  building" rule contradicted PRODUCT.md, which now says implementation is
  in scope).

## Open items flagged, not yet resolved (per the draft itself)

1. Grade 12 fixture off-by-one — needs re-derivation from the roster.
2. Teacher file rebuild status unclear — nine real surname-collision
   mismatches (ALDRIDGE, ALDWORTH, ALLY, BARNES, BHUNGANE, ELLIS, LAMB, VAN
   DEN BERG, VISSER) need resolution.
3. Anchor teacher ranking needs recompute from the rebuilt qualification
   file.
4. 253 vs 190 section count discrepancy in [[bounds-ladder]] — unreconciled.

**This page records that the two doc trees disagree; it does not resolve
which is authoritative.** That's a decision for Eugene/Claude Code in the
timemath repo itself, not something to silently pick a winner on here.
