# TimeExam — open questions

Grouped by whether they block implementation.

## Blocking — needed before the relevant module can be written

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
| `K` | Marking turnaround, days | 5? | Ask HODs what turnaround they actually work to |
| `R` | Candidates per invigilator | 30? | School/exam-board policy — a stated rule, not a tuned value |
| `SF` | Stress factor per subject, 1–5 | — | Teacher judgement per subject; this is a policy input |
| `MF` | Marking effort per script | `= SF` | Refine once someone times a batch |
| `λ_w` | Window weights, w = 2,3,4,5 | 3, 2, 1, ? | TimePyBling's proven 3/2/1; the w=5 weight is new |
| `α` | Fair-share slack multiplier | 1.0–1.2 | Tune until a "fair" schedule is achievable at all |
| `p` | Fairness exponent | 2 | Try 1 and 3 on real data and compare the worst-off student |
| — | Front-load weight (§4.4) | low | Raise until marking distribution is acceptable, watch week-1 student load |
| — | PM penalty (§4.3) | — | Set so PM sessions appear only when genuinely needed |
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

- **Seating capacity is out of scope.** No venue input file; slots are not
  constrained by seat counts. Invigilator demand derives from headcount and
  `R` instead (`DESIGN.md` §3.3).
- Paper key is `(subjectCode, grade, paperNo)` — per grade.
- Marking is consumed at a flat rate over `K` days.
- Session 2 is a cost, not a gate (`DESIGN.md` §4.3).
- Weeks are calendar structure, not processing order (§5).
- Invigilation is a post-placement matching, not part of the inner loop
  (§6).
- Student fairness is measured against a per-student baseline, not a global
  threshold (§4.1).
