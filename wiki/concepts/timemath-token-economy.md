---
tags: [concept, timemath, workflow]
created: 2026-07-26
updated: 2026-07-26
sources: [raw/sources/timemath/docs/TOKENS.md]
---

# Token economy — Claude Code sessions on timemath

Standing policy for [[timemath-project]] Claude Code sessions ("gremlin"),
not a wiki-maintenance rule. Chat is disposable, docs are the memory —
`/handoff` before anything that discards context.

* `/clear` after `/handoff` at any task boundary, at ~50% context, before a
  new phase — never mid-debug.
* Model switching is free: Opus/Fable for phase design and subtle bugs,
  Sonnet for day-to-day coaching, Haiku for mechanical sweeps. Switch down
  immediately after a design is agreed.
* Guided-coding mode itself is the biggest saver — Claude never writes large
  files, only diffs/snippets.
* Read files surgically (offset/limit, Grep) — never whole
  workbooks/modules.
* No subagents unless explicitly asked.
* Batch open questions into one message.
