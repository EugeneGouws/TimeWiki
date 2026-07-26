# CLAUDE.md — TimeWiki

Personal knowledge base for the timetable-solver project (`E:\timemath`) and
its legacy attempts (TimeEduSuite, earlier Python/CP-SAT tries, chat-side
brainstorm sessions). Built and maintained per the LLM Wiki pattern
(`llm_wiki.md` in this repo — read it if this file is ever unclear on
intent).

**Agents:** Claude Code (this CLI) and claude.ai Projects only. No
Cursor/Codex/AGENTS.md — do not create one.

## Layers

* **`raw/`** — immutable source material. `raw/sources/` holds source docs
  (brainstorm dumps, exported chat transcripts, old planning docs, legacy
  code snapshots as text). `raw/assets/` holds images. Never edit files here
  once added — a correction is a new source, not an edit to an old one.
* **`wiki/`** — everything the LLM writes and owns.
  * `wiki/entities/` — durable objects: people/teachers, subjects, file
    formats (`TT<year>.xlsx`, `Phase01_<year>.xlsx`), named legacy attempts
    (e.g. "TimeEduSuite", "solver-attempt-2026-07-precolour").
  * `wiki/concepts/` — recurring ideas that cut across sources: SDR/Hall's
    theorem, section-vs-lesson, band structure, precolouring, the exception
    rule, etc.
  * `wiki/summaries/` — one page per ingested source in `raw/sources/`.
  * `wiki/synthesis/` — cross-source answers, comparisons, standing theses.
    Query results worth keeping get filed here, not left in chat history.
* **`index.md`** — content catalog. One entry per entity/concept/synthesis
  page; summaries are covered by the source-to-summary filename rule stated
  in its Summaries section, not listed individually. Read first when
  answering a query.
* **`log.md`** — append-only timeline of ingests/queries/lints.

## Relationship to `E:\timemath`

`E:\timemath` is the live coding project (Python, guided-coding mode,
CLAUDE.md there is authoritative for *current build state*). TimeWiki is
where the *history* — prior attempts, dead ends, brainstorm sessions, and
things established but not currently active code — accumulates so it isn't
lost or re-litigated. When the two disagree on a settled fact, the timemath
repo's CLAUDE.md wins for anything about the current build; TimeWiki wins
for anything about "what did we try before and why did it fail."

Migration of `E:\timemath\docs` and `E:\timemath\solver-docs` into this wiki
is a deliberate future ingest pass, not automatic — do not silently pull
content over. Each doc gets read, summarized, and integrated properly
(entity/concept pages updated, cross-refs added), not just copy-pasted.

## Conventions

* **Frontmatter** on every wiki page:
  ```yaml
  ---
  tags: [entity|concept|summary|synthesis, <project>, ...more]
  created: YYYY-MM-DD
  updated: YYYY-MM-DD
  sources: [raw/sources/filename.md, ...]
  ---
  ```
  The second tag is the **project scope** — `timemath`, `timeedusuite`, or a
  new project's slug. Every page carries one unless it is genuinely
  cross-project. Keep it in sync with the namespace note in `index.md`.
* **Prose wraps hard at 76 columns.** Frontmatter, tables, headings, and
  fenced code are left unwrapped.
* **Links**: `[[wiki-relative name]]` Obsidian-style. Link liberally; a link
  to a page that doesn't exist yet marks something worth writing, not an
  error.
* **Legacy attempts get their own entity page**, not a shared "history"
  dump. Each failed/superseded approach should be traceable to *why* it
  failed, distinctly from the others — the whole point of this wiki is to
  stop re-discovering the same dead ends.
* **Contradictions are flagged, not silently overwritten.** If a new source
  contradicts an existing page, note both claims and which source is newer/
  more authoritative, rather than deleting the old claim outright.

## Workflows

**Ingest** (`raw/sources/<name>` added): read it, discuss key takeaways with
Eugene, write/update `wiki/summaries/<name>.md`, update every entity/concept
page it touches, update `index.md`, append a `## [date] ingest | <title>`
entry to `log.md`.

**Query**: read `index.md` first, drill into relevant pages, synthesize an
answer with citations to wiki pages (and sources where relevant). If the
answer is worth keeping, offer to file it into `wiki/synthesis/` and log it
as `## [date] query | <topic>`.

**Lint** (run when asked, not automatically): check for contradictions
between pages, claims superseded by newer sources, orphan pages with no
inbound links, concepts mentioned repeatedly but lacking their own page,
missing cross-references. Report findings; only fix on confirmation. Log as
`## [date] lint | <summary>`.

## Current state

Two ingest passes done (2026-07-26): 17 sources from `E:\timemath`
(repo-root CLAUDE.md × 2 snapshots — it changed mid-session, see
[[solver-docs-consolidation]] — plus `docs/*` ×5, `solver-docs/*` ×10) under
`raw/sources/timemath/`, namespaced to `timemath`; and 7 curated sources
from `E:\TimeEduSuite` + `E:\Rollover` (unprefixed, directly under
`raw/sources/`) covering the legacy production suite.
`E:\TimeEduSuite\data\` (real student/teacher records) deliberately excluded
from both — PII, out of scope. See `index.md` for the full page list and
`log.md` for all ingest/lint entries, including a concurrency reconciliation
(one duplicate stub page merged then removed, no content lost) and a live
contradiction flagged in `E:\timemath\CLAUDE.md` itself (unresolved by
design).

**Multi-agent note:** a separate agent is expected to populate TimeWiki from
other projects going forward. To avoid collision: tag every new page with
its project scope (see Conventions) and cross-reference across projects
where the concept isn't genuinely shared; check `index.md` before creating a
page that might already exist under a different name; prefer adding to an
existing page over creating a near-duplicate.
