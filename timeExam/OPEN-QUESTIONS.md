# TimeExam — open questions

Grouped by whether they block implementation.

## Blocking — needed before the relevant module can be written

**Who can invigilate, and how many are there?** Now the binding constraint
(`DESIGN.md` §1): rooms are ample, but every room needs a teacher who is
therefore not marking. Needed: the pool size, and whether non-teaching
staff, HODs or senior management count toward it. If the pool is materially
larger than the teaching staff, the constraint loosens and §4 has real
freedom; if it is the teaching staff alone, expect it to bind.

**Can a teacher invigilate during a period they would otherwise teach?**
Gr8–9 carry on with normal lessons in weeks 1–2 (`DESIGN.md` §1.2). If
invigilators must come only from teachers free in that period, supply is
much tighter than headcount suggests. `timetable.json` holds the teaching
commitments, but two mappings are needed to use them — see below.

**Bell times per day type, and the cycle-to-calendar mapping.** Teaching
commitments cannot be placed on the exam wall clock without both:
`timeslots` are bare labels with no times, and the teaching timetable runs
an 8-day cycle rather than a Mon–Fri week. TimeView already holds the times
(reference table, per day type: Normal / Assembly / Test / Long Reg), so
this is an extraction, not new data — but the cycle-to-calendar mapping for
the exam period still has to come from somewhere. Blocks §3.4.

**Gr8–9 sitting times.** Known to be later than 07:30/12:30, exact times
outstanding.

**Are venues really uniform at 25?** The design assumes every venue holds
exactly 25. If there is one hall holding 150 plus a set of classrooms, the
`rooms(p) = ceil(cohort/25)` model understates capacity — harmless for
rooms, which are slack, but it would *overstate* invigilator demand, which
is not. Worth confirming, since one invigilator per 25 candidates in a
150-seat hall means six invigilators in one room, not six rooms.

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
| — | Invigilator pool size | **needed** | See Blocking above; gates whether §4 matters |
| `K` | Marking turnaround, days | 5? | Ask HODs what turnaround they actually work to |
| `R` | Candidates per invigilator | **25** | Settled — one invigilator per venue |
| `SF` | Stress factor per subject, 1–3 | — | Teacher judgement; coarse on purpose — ties are stage-2 headroom |
| `MF` | Marking effort per script | `= SF` | Refine once someone times a batch |
| `λ_w` | Window weights, w = 2,3,4,5 | 3, 2, 1, ? | TimePyBling's proven 3/2/1; the w=5 weight is new. **Integers** |
| `α` | Fair-share slack | 1.0–1.2 | As an integer ratio `α_num/α_den`, never a float |
| `p` | Fairness exponent | 2 | Try 1 and 3, compare the worst-off student |
| — | Front-load weight (§4.4) | low | Raise until marking distribution is acceptable, watch week-1 student load |
| — | Session-2 fatigue (§4.3) | — | Fixed per band; sittings start at set clock times |
| — | Co-freedom weight (§4.5) | low | Watch that it does not starve invigilation cover |

**Weight calibration is part of the optimality claim, not decoration.**
Proving `Z` minimal proves nothing about fairness if `Z` mis-describes
stress (`DESIGN.md` §4.0.2) — the arithmetic is self-certifying, the
modelling is not. So the weights need to be defensible to whoever is shown
the schedule, which means:

- The penalty-breakdown view is a prerequisite, not a nicety
  (`PORT-NOTES.md`, inherited bug 2) — terms cannot be balanced against a
  single aggregate number.
- Only the **stage-B** weights are free to trade against each other. The
  stage-A student weights (`λ_w`, `α`, `p`) sit above the lexicographic
  split and set what "optimal" means, so they warrant more scrutiny than
  anything below.
- Keep every stage-A weight **integral** (§4.0.1). A rational weight breaks
  the exact `Z_stu = Z*` constraint that the whole two-stage guarantee
  rests on.

## Design questions, deferrable

**Link `timeedu_core` or stand alone?** Largely settled by the language
choice: a Python + CP-SAT solver cannot link the C++ core, so TimeExam
integrates at the JSON contract instead. Worth recording as a deliberate
deviation from the suite's "C++/Qt tool on the shared core" plan
(`REPO-SKELETON.md`), since the core's domain objects (lesson, timeslot,
placement) were never an obvious fit for exam objects (paper, sitting,
cohort) anyway.

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
- **Sessions are 40-minute periods**, two sittings per day, at fixed clock
  times **per grade band**: Gr12 08:00/13:00, Gr10–11 07:30/12:30, Gr8–9
  later (TBC). Bands overlap in wall-clock time (`DESIGN.md` §1.2).
- **Exam windows are nested and end together** — Gr10–12 weeks 1–6, Gr8–9
  weeks 3–6. The fairness baseline `D_s` is per band (§4.1).
- **Session 2 starts at a fixed time**, so a session-1 paper must fit the
  5-hour gap (≤ 3 h 45 writing). Hard constraint, not a cost.
- **Duration is per paper**, from the xlsx, and carries **+20 min per
  hour** of overhead. So occupancy = writing time × 4/3, and every
  exam-hour is exactly 2 periods (`DESIGN.md` §1.3).
- **Rooms are ample** — enough to seat every student at once, plus spares.
  The scarce resource is invigilators, not space (`DESIGN.md` §1).
- Paper key is `(subjectCode, grade, paperNo)` — per grade.
- Marking is consumed at a flat rate over `K` days.
- Session 2 is a cost, not a gate (`DESIGN.md` §4.3).
- Weeks are calendar structure, not processing order (§5).
- **Invigilation is a second phase** — a post-placement matching, never
  inside the search inner loop (§6).
- Student fairness is measured against a per-student baseline, not a global
  threshold (§4.1).
- **The schedule must be provably optimal**, so the primary method is exact
  (CP-SAT), not local search. A heuristic can supply incumbents and act as
  fallback, but never as the claim (§5).
- **Student and teacher objectives are lexicographic, not weighted.**
  Stage A minimises the student score to proven optimality; stage B pins
  `Z_stu = Z*` and optimises teachers within the tie set (§4.0).
- **`SF` stays 1–3.** Coarse on purpose: the ties it produces are the
  headroom stage B works in. An earlier draft argued for 1–5; that was
  wrong for this design (§1.3).
- **`Z_stu` is integer-only.** No floating point, or the `= Z*` constraint
  is unsafe (§4.0.1).
