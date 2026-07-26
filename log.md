# TimeWiki Log

Append-only. One entry per ingest / query-page / lint pass. Prefix format is
load-bearing — keep it exact so `grep "^## \[" log.md | tail -5` works.

## [2026-07-26] scaffold | Initial wiki structure

Created `raw/{sources,assets}`,
`wiki/{entities,concepts,summaries,synthesis}`, `index.md`, `log.md`,
`CLAUDE.md`. No content migrated yet — `E:\timemath\docs` and
`E:\timemath\solver-docs` remain the source of truth until a deliberate
ingest pass moves them in. Decision: no AGENTS.md (Claude Code + claude.ai
only, no Cursor/Codex).

## [2026-07-26] ingest | TimeEduSuite + Rollover crawl (6 raw sources, 20 wiki pages)

Crawled `E:\TimeEduSuite` (root docs, core/, TimeBuilder/, TimeEditor/,
TimeView/, tools/converter/, docs/archive/, Archived/) and `E:\Rollover` via
6 parallel haiku subagents, each writing one curated `raw/sources/*.md` file
(not raw dumps — filtered to architecture/decisions/lessons).
`E:\TimeEduSuite\data\` (real student/teacher records) deliberately excluded
— PII, out of scope. Build/.git/.idea artifacts also excluded.

Ingested the 6 raw sources into 9 entity pages (suite hub + core,
TimeBuilder, TimeEditor, TimeView [flagged stable/live, not legacy],
converter toolchain, and 3 legacy attempts: Rollover prototype, TimePyBling,
TimeVerify) and 4 concept pages (ejector/repair engine, JSON schema
contract, rollover-year-advancement, and a cross-cutting "Python prototype →
C++ rewrite" pattern spanning 3 of the legacy attempts). 7 summary pages
added, each pointing back to its raw source and forward to the
entity/concept pages it fed — kept thin to avoid restating content already
in the wiki layer. `index.md` updated.

Not yet touched: `E:\timemath\docs` / `solver-docs` migration (separate,
deliberate future pass per CLAUDE.md).

## [2026-07-26] ingest | timemath project docs (14 sources)

Ingested `E:\timemath\CLAUDE.md`, all 5 files in `docs/`, and all 8 files
under `solver-docs/` (CLAUDE.md, PLAN.md, CONSOLIDATION.md, explorations/×5,
archive/BRAINSTORM.md) as copies into `raw/sources/timemath/` — originals
left in place, still live/authoritative for the actual coding project. Wrote
1 project entity page (`timemath-project`), 2 supporting entity pages
(`timeedu-suite` stub, `solver-docs-consolidation`), 9 concept pages
(pipeline, basket-sdr, universal-block-and-band, independent-verification,
phase3/phase4/repair-model/warm-start-rejected, bounds-ladder,
token-economy), and 14 one-per-source summary pages. Flagged but did not
resolve: repo-root vs `solver-docs/` staging-draft divergence on CLAUDE.md
and PLAN.md (see `solver-docs-consolidation` — merge decision belongs to
Eugene/Claude Code in the timemath repo, not decided here). Scope
discipline: everything filed under a `timemath` path/tag prefix, per
Eugene's instruction that a separate agent will populate other projects into
this same wiki — check prefix before assuming a page's scope.

## [2026-07-26] lint | reconcile concurrent TimeEduSuite + timemath ingests

Both ingest passes above ran concurrently in separate sessions; no file
collisions (disjoint path prefixes) but one semantic duplicate: the timemath
session's `wiki/entities/timeedu-suite.md` stub was written *pending* a
TimeEduSuite ingest that the other session was doing at the same time.
Resolved: turned the stub into a redirect to `timeedusuite-suite.md`, moved
its firewall note (timemath doesn't reference TimeEduSuite code except
`ClashDetector`/[[independent-verification]]) onto the real hub page, and
rewrote `index.md` to merge both namespaces into one catalog. No content was
deleted, only redirected.

## [2026-07-26] lint | re-verify timemath docs against live repo

Other agent's timemath ingest session had finished; re-checked `E:\timemath`
against the wiki's snapshot to see if anything moved while both sessions
were active. Found `E:\timemath\CLAUDE.md` had changed since its first
snapshot: adopted the `solver-docs/` staging draft's TimeEduSuite-firewall
section and corrections 5–8 (dup Gr10 IDs, GOVENDER/WYLIE splits, stale 2026
teacher table, warm-start rejection) into its own prose — genuine new
established facts, promoted into [[timemath-project]]. But its own
next-steps list also claims `solver-docs/` was merged into `docs/` and
deleted; on disk `solver-docs/` still exists in full and `docs/PLAN.md` is
byte-identical to the pre-update snapshot. Added a second raw snapshot
(`CLAUDE_root_2026-07-26b.md`) and updated [[solver-docs-consolidation]] to
track this as a **live contradiction inside the project's own source of
truth** (not just an unmerged draft) — deliberately not resolved here;
flagged for Eugene/Claude Code to reconcile in the timemath repo itself.
`index.md` updated.

## [2026-07-26] lint | pre-commit consistency pass (6 fixes)

Full read of all 50 wiki pages plus `index.md` / `log.md` / `CLAUDE.md`.
Clean: zero broken `[[links]]`, complete frontmatter on every page, no
factual contradictions between pages beyond the one flagged by design
(`solver-docs-consolidation`). Fixed six consistency defects:

1. **Deleted `wiki/entities/timeedu-suite.md`** — the redirect stub left by
   the concurrency reconciliation above. Its content was already fully
   preserved on `timeedusuite-suite.md`, so it carried nothing but a
   forwarding pointer. Retargeted its one real inbound link
   (`summaries/timemath/solver-docs-CLAUDE.md`) to `[[timeedusuite-suite]]`
   and dropped it from `index.md`. Page count 50 → 49.
2. **`.gitkeep` in `raw/assets/` and `wiki/synthesis/`** — both were empty,
   so git would have dropped them on commit despite the scaffold entry
   above claiming they exist.
3. **Project tag added to 20 TimeEduSuite/Rollover pages** — they carried no
   project tag while every timemath page carried `timemath`, so tag-based
   scoping only worked for one namespace. Now `timeedusuite` is the second
   tag on all of them; convention written into `CLAUDE.md` and the
   `index.md` namespace note.
4. **`index.md` Summaries section rewritten** — it claimed the 24 summary
   pages were "listed above under Sources", but source names don't map
   visibly to summary filenames. Replaced with the explicit source →
   summary filename rule, including the two shortened names
   (`phase3-feasibility` → `explorations-phase3`, `phase4-teacher-min` →
   `explorations-phase4`). `CLAUDE.md`'s "one line per page" description of
   `index.md` amended to match what the index actually does.
5. **Source counts corrected in `CLAUDE.md`** — said 15 timemath + 6
   TimeEduSuite sources; on disk it is 17 + 7. The 2026-07-26 timemath
   ingest entry above under-counts for the same reason (its own prose lists
   `solver-docs/` as 8 files when it is 10, and the second CLAUDE.md
   snapshot was added later). Log entries left as written — append-only —
   the corrected count lives in `CLAUDE.md`.
6. **Uniform 76-column hard wrap** across all wiki pages, `index.md`,
   `log.md`, `CLAUDE.md` — the two ingest sessions had used different wrap
   conventions, which would have made every future diff noisy. Frontmatter,
   tables, headings, and fenced code left unwrapped; rule recorded under
   Conventions in `CLAUDE.md`.

Not fixed (by design): 24 summary pages have no inbound links. That is the
intended shape — summaries point forward to the entity/concept pages they
fed, and are reached via the source list, not via backlinks. Now stated
explicitly in `index.md` so a future lint doesn't re-flag them.
