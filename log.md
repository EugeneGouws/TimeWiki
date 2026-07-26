# TimeWiki Log

Append-only. One entry per ingest / query-page / lint pass. Prefix format is load-bearing — keep it exact so `grep "^## \[" log.md | tail -5` works.

## [2026-07-26] scaffold | Initial wiki structure

Created `raw/{sources,assets}`, `wiki/{entities,concepts,summaries,synthesis}`, `index.md`, `log.md`, `CLAUDE.md`. No content migrated yet — `E:\timemath\docs` and `E:\timemath\solver-docs` remain the source of truth until a deliberate ingest pass moves them in. Decision: no AGENTS.md (Claude Code + claude.ai only, no Cursor/Codex).
