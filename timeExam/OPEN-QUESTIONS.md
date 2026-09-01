# TimeExam — open questions

Grouped by whether they block implementation.

## Blocking — needed before the relevant module can be written

**How many exam venues (`V`)?** One number, and probably the most
consequential unknown in the whole design. At 25 candidates per venue, a
200-candidate grade paper needs 8 venues and 8 invigilators simultaneously.
If `V` is around 10, two large papers can never share a sitting and the
schedule is close to forced — which would make the §4 fairness objective a
tie-breaker rather than the main event, and would change where the
implementation effort should go (construction and feasibility, not weight
tuning). Get this before building §4.

**Are venues really uniform at 25?** The design assumes every venue holds
exactly 25. If there is one hall holding 150 plus a set of classrooms, the
`rooms(p) = ceil(cohort/25)` model understates capacity badly and a venue
list becomes necessary after all.

**Does "exam times in the xlsx" mean duration or fixed sittings?** Read as
duration (`DESIGN.md` §1). If a substantial number of papers arrive with
dates and times already fixed — externally set national papers, say — then
most of the schedule is given and the tool's job shifts from optimisation
to filling gaps around pinned papers.

**`timetable.json` v3.1 schema delta.** TimeWiki documents v2.1/v3.0 only.
Confirm against the live schema before writing the reader — particularly
whether `student_slots` is still normative and whether `Lesson.subjectCode`
/ `instanceIndex` are unchanged. Blocks `io/timetable_reader`.

**Exam xlsx format.** Undefined. What columns, one row per paper or per
grade-subject, how paper numbers are expressed, how linked papers are
marked. Blocks `io/paper_reader`.

**Does `markers(p)` split the cohort or pool the department?** If Maths Gr10
is taught by three teachers, does each mark their own students
(`scripts(t,p)` = their class size) or does the department pool and split
evenly? This changes the marking-load model materially — pooling makes
marking load nearly uniform within a department, splitting makes it track
teaching allocation. `DESIGN.md` §3.2 currently assumes splitting.

## Constants — need real data, not decisions

None of these can be guessed well; all should start at the stated seed and
be tuned against real output.

| Symbol | What | Seed | How to settle |
|---|---|---|---|
| `V` | Exam venues available | **needed** | Count them — see Blocking above; gates whether §4 matters |
| `K` | Marking turnaround, days | 5? | Ask HODs what turnaround they actually work to |
| `R` | Candidates per invigilator | **25** | Settled — one invigilator per venue |
| `SF` | Stress factor per subject, 1–5 | — | Teacher judgement per subject; this is a policy input |
| `MF` | Marking effort per script | `= SF` | Refine once someone times a batch |
| `λ_w` | Window weights, w = 2,3,4,5 | 3, 2, 1, ? | TimePyBling's proven 3/2/1; the w=5 weight is new |
| `α` | Fair-share slack multiplier | 1.0–1.2 | Tune until a "fair" schedule is achievable at all |
| `p` | Fairness exponent | 2 | Try 1 and 3 on real data and compare the worst-off student |
| — | Front-load weight (§4.4) | low | Raise until marking distribution is acceptable, watch week-1 student load |
| — | Session-2 fatigue (§4.3) | — | Scale from session 1's finish time, not a flat constant |
| — | Co-freedom weight (§4.5) | low | Watch that it does not starve invigilation cover |

**Tuning needs the penalty-breakdown view** (`PORT-NOTES.md`, inherited bug
2). Six weighted terms cannot be balanced against a single aggregate number.
Build the breakdown before attempting to tune.

## Design questions, deferrable

**Link `timeedu_core` or stand alone?** The core supplies domain model and
JSON I/O — real leverage — but couples TimeExam's release to core
versioning, and the core is built around *timetabling*, whose domain objects
(lesson, timeslot, placement) are not obviously the right ones for
*examining* (paper, session, cohort). Possible middle path: use the core's
JSON reader only, keep the exam model native.

**Is CVaR worth it?** `DESIGN.md` §4.1 suggests optionally restricting the
student term to the worst 10% of students. Try only after the basic version
produces sane schedules.

**Are cross-grade papers common or exceptional?** The locked per-grade key
makes a cross-grade paper into N linked objects. Fine if rare, verbose if
routine. If routine, revisit the key.

**Does the horizon include weekends?** `DESIGN.md` assumes Mon–Fri exam
days, but the *marking backlog* `B_t(d)` arguably continues over weekends —
teachers do mark on Saturdays. Whether `K` counts calendar days or exam
days changes the backlog curve.

**Study leave / non-exam days.** Are there days inside the horizon with no
exams that still count toward window spacing? A student's 5-day window
crossing a study day is materially different from one that does not.

**Should `SF` vary by grade?** A Gr12 paper carries more weight than a Gr8
one of the same subject. Currently `SF` is per subject only.

## Explicitly settled — do not relitigate without reason

- **Venues are uniform at 25 pax, one invigilator each.** So venues and
  invigilators are the same count: `rooms(p) = ceil(|cohort(p)| / 25)`.
  No venue file — constants only (`DESIGN.md` §1).
- **Sessions are 40-minute periods**, two sittings per day; a paper
  occupies `ceil(duration / 40)` consecutive periods (`DESIGN.md` §2.4).
- Paper key is `(subjectCode, grade, paperNo)` — per grade.
- Marking is consumed at a flat rate over `K` days.
- Session 2 is a cost, not a gate (`DESIGN.md` §4.3).
- Weeks are calendar structure, not processing order (§5).
- **Invigilation is a second phase** — a post-placement matching, never
  inside the search inner loop (§6).
- Student fairness is measured against a per-student baseline, not a global
  threshold (§4.1).
