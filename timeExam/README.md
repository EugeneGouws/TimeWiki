# TimeExam — design staging

**Status:** design only. No code, no repo yet.

TimeExam is the planned exam-scheduling tool for the TimeEduSuite family:
it imports a school timetable plus a list of papers to be written, and
produces a balanced exam schedule — balanced for students (writing load),
for teachers (marking load), and for the school (invigilation cover).

It replaces **TimePyBling**, the archived Python/Tkinter attempt
(`E:\TimeEduSuite\Archived\TimePyBling`, abandoned 2026-06-02 on
performance grounds). It is **not** a straight port: the scoring model here
is materially different from TimePyBling's. See `PORT-NOTES.md` for what
survives and what does not.

## Why this directory exists

This is **temporary staging inside TimeWiki**, not a permanent home.
TimeExam is about to become its own repository; these docs exist to seed it.

Deliberately *not* done, to avoid unpicking later:

- nothing filed into `wiki/entities/`, `wiki/concepts/` or `raw/sources/`
- `index.md` and `log.md` untouched

Once TimeExam settles into its own repo, a normal TimeWiki ingest pass can
file the settled outcome properly, and this directory goes away.

## Reading order

1. **`DESIGN.md`** — the specification. Inputs, data model, the three
   scores, the objective function, search strategy, invigilation. Start
   here.
2. **`PORT-NOTES.md`** — what carries over from TimePyBling, what is
   deliberately dropped, and the inherited bug list.
3. **`REPO-SKELETON.md`** — proposed layout for the new repository.
4. **`OPEN-QUESTIONS.md`** — everything still undecided, including the
   constants that need real data before they can be set.

## Decisions locked so far

| Decision | Choice |
|---|---|
| Paper key | `(subjectCode, grade, paperNo)` — strictly per grade |
| Marking turnaround | Flat rate over K days following the paper |

Everything else is open; see `OPEN-QUESTIONS.md`.
