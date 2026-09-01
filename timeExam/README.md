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

1. **`DESIGN.md`** — the specification. Inputs, data model, hard
   constraints, the three scores, the objective, solving strategy,
   invigilation, and defensibility. Start here.
2. **`LITERATURE.md`** — where this design sits against published exam-
   timetabling practice, the citations behind it, and the vocabulary that
   makes it read as standard. Read before defending anything to anyone.
3. **`PORT-NOTES.md`** — what carries over from TimePyBling, what is
   deliberately dropped, and the inherited bug list.
4. **`REPO-SKELETON.md`** — proposed layout for the new repository.
5. **`OPEN-QUESTIONS.md`** — everything still undecided, including the
   constants that need real data before they can be set.

## Decisions locked so far

| Decision | Choice |
|---|---|
| Paper key | `(subjectCode, grade, paperNo)` — strictly per grade |
| Marking turnaround | Flat rate over K days following the paper |
| Optimisation | Exact (CP-SAT), lexicographic: students, then teachers |
| `SF` | Integer 1–3, an institutional weighting — coarse on purpose |
| Fairness baseline | Per student, against their own band window |

Everything else is open; see `OPEN-QUESTIONS.md`.

## Resume here

For a cold start (new session, new machine, someone else entirely):

- **Branch:** `claude/timeexam-repo-scope-n8kh2l` in the TimeWiki repo.
  Everything is committed and pushed; nothing of value exists only in a
  chat log.
- **State:** design complete and internally consistent. No code written, no
  repo created yet.
- **Read:** `DESIGN.md` end to end, then `OPEN-QUESTIONS.md` for what is
  still unsettled. `LITERATURE.md` before defending anything to anyone.
- **Next action:** the blocking unknowns in `OPEN-QUESTIONS.md` are data
  questions for the school, not design questions — invigilator pool size,
  the 8-day-cycle to calendar mapping, Gr8–9 sitting times, the exam xlsx
  format, the v3.1 schema delta. Most of the design cannot be validated
  until those land.
- **First build step** once they do: `REPO-SKELETON.md` build order, step 2
  — the staffing diagnostic. It is the gate that determines whether the
  fairness objective has room to operate at all.

**Reversals recorded on purpose.** The spec argues against three of its own
earlier drafts — SF widened to 1–5 then returned to 1–3, C++ then Python,
venues then invigilators as the binding constraint. Those passages are
deliberate: each records why the first answer was wrong, so the reasoning
is not re-litigated. Do not tidy them into bare conclusions.
