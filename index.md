# TimeWiki Index

Catalog of every page in the wiki. Update on every ingest, query-page, or
lint pass. Read this first when answering a question — drill into pages from
here, don't grep blind.

**Namespace note:** two projects ingested so far. Every wiki page carries its
project as the second frontmatter tag — `timemath` or `timeedusuite` — so
scope is readable without opening `sources:`. Don't duplicate entity/concept
pages across projects without checking here first.
- `timemath/` — the active solver project (`E:\timemath`), ingested
  2026-07-26.
- `timeedusuite-*` / `rollover.md` (unprefixed, no subfolder) — the legacy
  production suite (`E:\TimeEduSuite`) + standalone `E:\Rollover` prototype,
  crawled 2026-07-26.

## Sources (raw/sources/)

**timemath/** — `CLAUDE_root.md` (first snapshot);
`CLAUDE_root_2026-07-26b.md` (second snapshot, same day — root CLAUDE.md
changed mid-session; claims a `solver-docs/` merge the filesystem
contradicts, see [[solver-docs-consolidation]]);
`docs/{PRODUCT,PLAN,HANDOFF,TOKENS,BRAINSTORM}.md`;
`solver-docs/{CLAUDE,PLAN,CONSOLIDATION}.md` (staging draft — still present
on disk despite root CLAUDE.md's claim otherwise);
`solver-docs/docs/explorations/{README,bounds-ladder,phase3-feasibility,phase4-teacher-min,repair-model,warm-start}.md`;
`solver-docs/docs/archive/BRAINSTORM.md`

**TimeEduSuite + Rollover** — `timeedusuite-core.md`,
`timeedusuite-archived.md`, `timeedusuite-timebuilder.md`,
`timeedusuite-timeeditor.md`, `timeedusuite-timeview.md`,
`timeedusuite-tools-and-archive.md`, `rollover.md`. `E:\TimeEduSuite\data\`
(real student/teacher records) deliberately **not** crawled — PII, out of
scope.

## Entities (wiki/entities/)

**timemath** — [timemath-project.md](wiki/entities/timemath-project.md) (the
solver project: status, pipeline, key facts) ·
[solver-docs-consolidation.md](wiki/entities/solver-docs-consolidation.md)
(**live contradiction**: root CLAUDE.md claims solver-docs/ merged+deleted,
filesystem disagrees — unresolved by design, flagged for the timemath repo
itself)

**TimeEduSuite** —
[timeedusuite-suite.md](wiki/entities/timeedusuite-suite.md) (hub:
three-tier architecture, component status table, timemath firewall note) ·
[timeedusuite-core-library.md](wiki/entities/timeedusuite-core-library.md)
(shared C++ engine — note: named `-library` to disambiguate from the summary
page of the same raw source, `timeedusuite-core`) ·
[timebuilder.md](wiki/entities/timebuilder.md) ·
[timeeditor.md](wiki/entities/timeeditor.md) ·
[timeview.md](wiki/entities/timeview.md) (**stable, live in production**) ·
[converter-toolchain.md](wiki/entities/converter-toolchain.md) ·
[rollover-cpp-prototype.md](wiki/entities/rollover-cpp-prototype.md),
[timepybling.md](wiki/entities/timepybling.md),
[timeverify.md](wiki/entities/timeverify.md) (legacy attempts)

## Concepts (wiki/concepts/)

**timemath** —
[timetable-solver-pipeline.md](wiki/concepts/timetable-solver-pipeline.md) ·
[basket-sdr.md](wiki/concepts/basket-sdr.md) ·
[universal-block-and-band.md](wiki/concepts/universal-block-and-band.md) ·
[independent-verification.md](wiki/concepts/independent-verification.md)
(imports `ClashDetector` from TimeEduSuite) ·
[phase3-column-student-assignment.md](wiki/concepts/phase3-column-student-assignment.md)
· [phase4-teacher-assignment.md](wiki/concepts/phase4-teacher-assignment.md)
· [repair-model.md](wiki/concepts/repair-model.md) ·
[warm-start-rejected.md](wiki/concepts/warm-start-rejected.md) ·
[bounds-ladder.md](wiki/concepts/bounds-ladder.md) ·
[timemath-token-economy.md](wiki/concepts/timemath-token-economy.md)

**TimeEduSuite** —
[ejector-repair-engine.md](wiki/concepts/ejector-repair-engine.md) (Phase C
repair spine, monotone-Φ) ·
[json-timetable-schema.md](wiki/concepts/json-timetable-schema.md)
(`timetable.json` contract, `instanceIndex`) ·
[rollover-year-advancement.md](wiki/concepts/rollover-year-advancement.md) ·
[python-prototype-to-cpp-pattern.md](wiki/concepts/python-prototype-to-cpp-pattern.md)
(3-instance recurring pattern)

## Summaries (wiki/summaries/)

One page per ingested source (24 pages: 17 timemath + 7
TimeEduSuite/Rollover), not listed individually — derive the filename from
the source listed above under Sources:

- **timemath** — `raw/sources/timemath/<dir>/<NAME>.md` →
  `wiki/summaries/timemath/<dir>-<NAME>.md`, where `<dir>` is the last path
  segment before the file (`docs/BRAINSTORM.md` → `docs-BRAINSTORM.md`;
  `solver-docs/docs/explorations/warm-start.md` →
  `explorations-warm-start.md`; `solver-docs/docs/archive/BRAINSTORM.md` →
  `archive-BRAINSTORM.md`). Root-level `CLAUDE_root*.md` keep their names
  verbatim. Two exceptions where the source name was shortened:
  `phase3-feasibility` → `explorations-phase3`, `phase4-teacher-min` →
  `explorations-phase4`.
- **TimeEduSuite/Rollover** — `raw/sources/<NAME>.md` →
  `wiki/summaries/<NAME>.md`, same name.

Every summary is thin, pointing to the entity/concept pages it fed rather
than restating content. Summaries are deliberately orphans (no inbound
links) — they point forward only.

## Synthesis (wiki/synthesis/)

_None yet._ (cross-source answers, comparisons, standing theses — filed back
from queries, not just ingests)

## Legacy attempts

- [[timepybling]] — abandoned 2026-06-02, Python/Tkinter perf-blocked,
  awaiting C++ port (TimeExam)
- [[timeverify]] — abandoned 2026-06-02, fully superseded by [[timeeditor]]
- [[rollover-cpp-prototype]] — standalone earlier prototype, absorbed into
  [[timeeditor]]'s rollover pipeline
- The five prior failed timemath attempts referenced in timemath's
  CLAUDE.md/PRODUCT.md are not yet separately documented — no source
  material for them located/ingested yet.
