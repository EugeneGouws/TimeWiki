# CLAUDE.md — TimeWiki

Personal knowledge base for the timetable-solver project (`E:\timemath`) and
its legacy attempts (TimeEduSuite, earlier Python/CP-SAT tries, chat-side
brainstorm sessions). Built and maintained per the LLM Wiki pattern
(`llm_wiki.md` in this repo — read it if this file is ever unclear on intent).

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
* **`index.md`** — content catalog, one line per page. Read first when
  answering a query.
* **`log.md`** — append-only timeline of ingests/queries/lints.

## Relationship to `E:\timemath`

`E:\timemath` is the live coding project (Python, guided-coding mode,
CLAUDE.md there is authoritative for *current build state*). TimeWiki is
where the *history* — prior attempts, dead ends, brainstorm sessions, and
things established but not currently active code — accumulates so it isn't
lost or re-litigated. When the two disagree on a settled fact, the timemath
repo's CLAUDE.md wins for anything about the current build; TimeWiki wins for
anything about "what did we try before and why did it fail."

Migration of `E:\timemath\docs` and `E:\timemath\solver-docs` into this wiki
is a deliberate future ingest pass, not automatic — do not silently pull
content over. Each doc gets read, summarized, and integrated properly
(entity/concept pages updated, cross-refs added), not just copy-pasted.

## Conventions

* **Frontmatter** on every wiki page:
  ```yaml
  ---
  tags: [entity|concept|summary|synthesis, ...more]
  created: YYYY-MM-DD
  updated: YYYY-MM-DD
  sources: [raw/sources/filename.md, ...]
  ---
  ```
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

Scaffold only (2026-07-26). No sources ingested. `index.md` and `log.md`
reflect this — update both as soon as the first ingest happens.
