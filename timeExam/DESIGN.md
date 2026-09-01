# TimeExam — design specification

Draft 1, 2026-09-01. Supersedes the TimePyBling scoring model.

## §0 Goal, and what changed from the brief

**Goal:** produce an exam schedule that is *as fair as possible for as many
students as possible*, while keeping teacher marking load survivable and
invigilation covered.

That goal is a **coverage** objective, not a sum-minimisation one, and it
drives most of the design decisions below. Five changes from the original
brief, each with its reason, plus one constraint the brief did not
anticipate:

1. **`students × SF` is split into three separate quantities.** Cohort size
   drives marking effort and invigilation demand; it has no bearing on how
   stressed an individual student is. Collapsed into one number, the "two
   conflicting heuristics" are actually one quantity and a cohort-scaled
   multiple of it — they cannot genuinely trade off. See §3.
2. **SF splits into SF (write stress) and MF (marking effort).** Maths P1
   is high-stress to write and fast to mark; History is moderate to write
   and slow to mark. One number cannot carry both, and these are exactly
   the two axes the conflicting heuristics live on. Default `MF = SF` where
   unmeasured.
3. **Fairness is measured per student against their own baseline**, not
   against a global threshold. A student taking 9 subjects has a
   structurally higher load than one taking 6; judged against one global
   threshold, the heavy student is permanently over it and the light one
   never near it, so the optimiser works only for the heavy students. See
   §4.1 — this is the single most important correction for the stated goal.
4. **The "spread one student's excess over others" mechanic is replaced by
   a convex penalty.** `max(0, excess)²` already prefers two students
   slightly over to one student badly over. Same behaviour, no
   redistribution bookkeeping, no special cases.
5. **Invigilation is a second stage, not part of placement.** Given a fixed
   placement it is a min-cost bipartite matching, solvable exactly and
   cheaply; evaluating it inside the local-search inner loop would be
   ruinous. See §6.

And one thing the brief did not anticipate:

6. **Invigilator supply is the binding constraint.** Rooms are ample —
   enough to seat the school at once — but every room in use needs a
   teacher who is therefore not marking. One invigilator per 25 candidates
   means seating a whole grade occupies eight teachers for the paper's full
   length. The §4 objective is really negotiating over teacher-time, not
   space. See §1 and §3.3.

### The structural fact that shapes the algorithm

**A learner belongs to exactly one grade, so the student objective is
separable by grade.** A student writes only their own grade's papers, so
their load, their windows and their fair-share baseline involve no other
grade. The dominant term therefore decomposes into independent per-grade
subproblems.

**The only cross-grade coupling is teachers** — marking and invigilation —
**and that is explicitly weighted below the student score.**

This gives a **strict priority order**, not a set of weights to blend:

| Rank | Term | Scope |
|---|---|---|
| Hard | Invigilator availability, rooms ≤ `V` | Cross-grade, wall clock |
| **Stage A** | Student fairness (§4.1) | **Per grade, separable** |
| **Stage B** | Marking, invigilation, front-load, co-freedom (§4.2–4.5) | Cross-grade |

Stage B optimises **only within the set of stage-A optima** — teacher needs
are filtered in afterwards without changing the student score (§4.0). The
schedule must also be **provably optimal**, which rules out local search as
the primary method and points at CP-SAT (§5).

Two conflicts in the brief that were not stated, and are now explicit
weighted terms rather than rules (§4.4, §4.5):

- **"Heaviest marking earlier" fights student fairness.** The biggest
  cohorts are the universal subjects — Maths, English, Home Language —
  which nearly every student writes. Front-loading them spikes student load
  in week 1.
- **"Whole department free together after their paper" fights invigilation
  cover.** If all the Maths teachers are free, someone else is invigilating.
  That someone is systematically a small-subject teacher.

---

## §1 Inputs

| Input | Source | Notes |
|---|---|---|
| Timetable | `timetable.json` **v3.1** | Student/teacher/lesson data |
| Papers | xlsx | Papers per grade per subject, plus **duration** each |
| Constants | `constants/*` | `SF`, `MF` per subject; venue and session constants |

**Schema version caution.** TimeWiki documents `timetable.json` at v2.1/v3.0
(`wiki/concepts/json-timetable-schema.md`). v3.1 is newer than anything
ingested here, so the delta is unknown. Confirm the live schema before
implementing the reader; do not assume the documented shape is current.

**Venues are uniform, so no venue file is needed** — just constants:

| Constant | Value | Meaning |
|---|---|---|
| `VENUE_PAX` | 25 | Maximum candidates per venue |
| `R` | 25 | Candidates per invigilator — one per venue |
| `PERIOD` | 40 min | Session granularity |
| `OVERHEAD` | +20 min / hour | Added to writing time — see §1.3 |
| `V` | ample | Rooms for every student at once, plus spares |

Because `R == VENUE_PAX`, invigilators and venues are the same count: a
paper needs `ceil(|cohort| / 25)` of each, simultaneously, for its whole
occupancy.

> **Rooms are not scarce; invigilators are.** `V` is large enough to seat
> every student simultaneously, so the room constraint is slack almost
> everywhere. But every room in use needs a teacher standing in it, and
> that teacher cannot be marking (§3.2). Seating the whole school at once
> would take `ceil(N_students / 25)` invigilators — for 1000 students, 40
> teachers occupied at the same moment. **Invigilator supply is the binding
> constraint of this design**, and it is what the §4 objective is really
> negotiating against.

Rooms are still counted (§3.3) — they are what *determines* invigilator
demand — but `rooms(slot) ≤ V` becomes a cheap guard rather than the
dominant gate.

### §1.2 Grade bands and sitting times

Sittings are **per grade band, at fixed clock times set by policy** — not a
single school-wide schedule:

| Band | Session 1 | Session 2 |
|---|---|---|
| Gr 12 | 08:00 | 13:00 |
| Gr 10–11 | 07:30 | 12:30 |
| Gr 8–9 | later start (times TBC) | |

**Season structure: the windows are nested, ending together.** Seniors
write for more weeks; juniors start later; **everyone finishes on the same
day**. The worked example to design against:

```
week:      1     2     3     4     5     6
Gr 10–12  [=====================================]   6 weeks
Gr  8–9               [=======================]     4 weeks
           ^ seniors only ^      ^ both bands writing ^
```

So `D_senior ≈ 30` exam days and `D_junior ≈ 20` — which is why the
fairness baseline is per band (§4.1).

Note the proportions: the overlap phase is **four of the six weeks**. The
senior-only relief is the minority of the season, not the norm, so the
concurrent-demand case is the one to design for.

Four consequences, each load-bearing:

**1. Bands overlap in wall-clock time.** Gr12 writing 08:00–12:00 runs
alongside Gr10 writing 07:30–11:30. They are different slots but they
compete for the same invigilators at the same moment. Invigilator demand
therefore **cannot** be summed per slot — see §3.3.

**2. The period grids do not align across bands.** A 30-minute offset
against a 40-minute period means Gr10's boundaries fall at 07:30, 08:10,
08:50… and Gr12's at 08:00, 08:40, 09:20… Periods remain a clean unit
*within* a band; across bands, only wall-clock time is common. Use minutes
for anything that spans bands.

**3. Session 2's start is fixed, not derived.** Because the start times are
policy, a long session-1 paper does not push session 2 later — it must
simply fit the 5-hour gap, i.e. `duration ≤ 5 h × 3/4 = 3 h 45`.

**Max paper length is 3 hours**, so this always holds: 180 min of writing
is 240 min of occupancy against 300 available, leaving an hour spare. The
check is therefore an **import-time validation** (reject a paper over 3 h
as bad data), not a live placement constraint — the search never needs to
test it. The second-sitting penalty (§4.3) is a fixed per-band fatigue
constant, not a function of session 1's contents.

**4. Invigilator supply and demand both jump when the juniors start.** The
season has two regimes:

| Phase | Juniors are | Junior teachers | Papers to staff |
|---|---|---|---|
| Weeks 1–2 | in class | **unavailable** (teaching) | seniors only |
| Weeks 3–6 | writing | available | seniors **and** juniors |

Supply rises exactly when demand does, and which phase is tighter cannot be
settled from first principles — weeks 1–2 have lower demand but a reduced
pool; weeks 3–6 have the full pool but more papers. Measure it. This is the
first thing the step-2 diagnostic (`REPO-SKELETON.md`) should report, **per
phase rather than averaged** — an average over the season hides the binding
case completely.

**One genuine synergy, worth exploiting.** Placing heavy senior papers in
weeks 1–2 both avoids competing with junior papers *and* satisfies
"heaviest marking earlier" (§4.4). Two objectives pulling the same way is
rare in this design — but it still trades against senior student fairness
(§4.1), since cramming senior papers into two weeks spikes exactly the load
the fairness term exists to flatten. Tune the front-load weight with this
specific trade-off in view.

### §1.3 Duration and occupancy

The xlsx gives each paper's **writing time**. A paper occupies its venue
for longer than that:

```
occupancy(p) = duration(p) × 4/3        // +20 min per hour
periods(p)   = occupancy(p) / 40
```

This lands exactly on the period grid — **every exam-hour is 2 periods**:

| Writing time | Occupancy | Periods |
|---|---|---|
| 1 h | 80 min | 2 |
| 1½ h | 120 min | 3 |
| 2 h | 160 min | 4 |
| 3 h | 240 min | 6 |

Any duration in whole half-hours yields a whole number of periods, which is
almost certainly why the period is 40 minutes. No rounding logic is needed
unless papers arrive with odd durations — validate on import and reject or
round up explicitly rather than silently.

**Use the right one of the two.** Occupancy drives venue and
invigilator-period commitment (§3.3, §6). Writing time drives student
fatigue (§4.3). Conflating them overstates student load on long papers and
understates invigilator cost on short ones.

**Assumption to confirm:** "exam times in the xlsx" is read here as each
paper's **duration**, not a pre-assigned date and time. If some papers
arrive with fixed sittings, they are `pinned` (§2) and the rest schedule
around them — but if *most* arrive fixed, this is not a scheduling problem
and the design changes fundamentally.

### Constants file

Per subject: `SF`, `MF`. Duration is **per paper**, from the xlsx — two
papers of the same subject routinely differ in length.

**`SF` — student stress factor, integer 1–3.** Kept coarse **deliberately**.

An earlier draft of this spec argued for widening to 1–5, on the grounds
that ties give a search nothing to discriminate between. That was wrong for
this design. Under lexicographic optimisation (§4.0), ties in the student
score are not a defect — they are the **feasible region the teacher stage
gets to work in**. A student objective fine enough to order every schedule
uniquely would leave the second stage nothing to choose between, and
teacher needs would be unimprovable without damaging student fairness.
Coarse SF is what buys the headroom.

**The honest trade-off:** a tie asserts that two schedules are *equally
fair*, and at 1–3 that assertion is coarse — a brutal subject and a
moderate one may land on the same level, so genuine unfairness can hide
inside a tie. This is a deliberate choice of teacher headroom over fairness
resolution, not a free lunch. If real schedules come out visibly unfair
between tied options, the lever is a finer SF, at the cost of stage-2 room.

`SF` is a per-subject constant, so a student's total stress is fully
determined by their subject choices. What `SF` deliberately does *not*
capture is that two SF-3 subjects back to back are worse than their sum —
that is the job of the consecutive-day convolution in §4.1. The two
mechanisms together are what express "fair".

**`MF` — marking effort per script.** Independent of `SF`.

---

## §2 Data model

### Paper

```
Paper {
  subjectCode, grade, paperNo    // identity — the key
  cohort        : StudentBitset  // students who write it
  markers       : [TeacherId]    // who marks it
  duration, SF, MF
  linkGroup     : LinkGroupId?   // §2.4
  pinned        : Slot?          // fixed placement, if any
}
```

**Key is `(subjectCode, grade, paperNo)`** — strictly per grade (locked
decision).

*Accepted cost of this choice:* a paper sat by two grades simultaneously —
which TimePyBling supported natively as a "cross-grade exam schedule" —
becomes N separate `Paper` objects bound by a hard link group. The
machinery for this already exists for the two-sitting linkage case (§2.4),
so the cost is modelling verbosity, not new code. If cross-grade papers
turn out to be common rather than exceptional, revisit the key.

### Cohort derivation

The exam cohort is a *subject enrolment*, not a timetabled lesson roster: a
student in `MA_SMITH_10_1` and one in `MA_JONES_10_2` write the same Maths
paper. So:

> cohort(subject, grade) = union of students across **all lesson instances**
> sharing that `(subjectCode, grade)`

Read from `student_slots` — normative per the schema — joined via
`Lesson.subjectCode` and `Lesson.instanceIndex`.

**Never parse lesson labels.** `CODE_SURNAME_GG[_N]` looks parseable and is
not; the suite added real fields (`Subject.code`, `Lesson.subjectCode`,
`Lesson.instanceIndex`) specifically to kill string-parsing, and the shared
repair-engine contract forbids it. See `wiki/concepts/json-timetable-schema.md`,
"the field that killed string-parsing". This is a documented recurring
source of bugs across the suite.

### Cohorts as bitsets

Store `cohort` as a fixed-width bitset over student IDs.

This is not an optimisation detail — it is the reason a C++ build succeeds
where the Python one failed. It makes a placement move cost `O(|cohort|)`
instead of `O(|students|)`, and it makes student-overlap tests a single
bitwise AND. The pattern is already proven in this codebase: the Rollover
prototype used a "bitset of enrolled students (efficient overlap
detection)" for exactly this purpose (`raw/sources/rollover.md:20`).

### Calendar and link groups

- Horizon: an ordered list of exam **days**, Mon–Fri, grouped into weeks.
- Each day is a grid of **40-minute periods**.
- A **sitting** starts on a period boundary. Two sittings per day: session 1
  and session 2.
- `Slot = (day, sitting)`. A paper placed in a sitting occupies
  `periods(p)` consecutive periods from that sitting's start, where
  `periods(p)` is computed from occupancy, not raw writing time (§1.3).
- Week index derives from day index; weeks are a *structure*, never a
  processing order (§5).

**Sittings start together, papers end apart.** All papers in a sitting begin
at the same moment; each runs for its own duration. This is how exam
sittings actually work, and it has a consequence worth stating:

> **Session 2 starts at a fixed clock time** (§1.2), so a long session-1
> paper does not push it later — the paper must simply fit in the gap.
> With 5 hours between sittings, that caps a session-1 paper at 3 h 45 of
> writing time. Check it on placement; it is a hard constraint, not a cost.

**Invigilator and venue commitment is measured in periods**, which finally
gives invigilation a unit: a teacher watching a 3-hour paper is committed
for 6 periods — three times a 1-hour paper's 2 — and that is what trades
off against their marking backlog.

**Link group** — a set of papers that must be placed together with a fixed
internal arrangement. Covers three cases with one mechanism:

- two papers on the same day, session 1 + session 2 (the brief's explicit
  requirement)
- a cross-grade paper split across grades by the per-grade key (§2.1)
- pinned study blocks

Link groups are **composite placeable units**: the optimiser moves the whole
group atomically. A move can therefore never break a link, and no repair
pass is needed for them. This is strictly better than treating linkage as a
constraint that gets violated and then fixed.

---

## §3 The three scores

Replacing the single `students × SF`.

### 3.1 Student writing load — per student, per day

```
L_s(d) = Σ  SF(p)        for p written by s on day d
```

**Cohort-size independent.** A student is not more stressed because 200
others sit the same paper.

### 3.2 Marking load — per teacher, per day

```
M(t,p) = scripts(t,p) × MF(p)
```

where `scripts(t,p)` is that marker's own share of the cohort.

Marking is consumed at a **flat rate over K days** following the paper
(locked decision), giving a backlog curve:

```
B_t(d) = Σ  M(t,p)/K     over papers p marked by t
                          with  day(p) ≤ d < day(p) + K
```

`B_t(d)` is what makes "cannot mark and invigilate" a well-defined
statement: without a turnaround model the constraint has no meaning, since
marking is not an instantaneous event tied to the exam session.

### 3.3 Venue and invigilator demand — per slot

Demand is **per paper**, then summed over a wall-clock instant:

```
rooms(p) = ceil(|cohort(p)| / 25)            // venues == invigilators
```

**Two ways to get this wrong, both of which produce unstaffable schedules:**

**1. Do not compute `ceil(Σ|cohort| / 25)`.** Papers cannot share a venue,
so two papers of 10 candidates need two rooms, not one. Sum the per-paper
ceilings; ceiling the sum understates demand.

**2. Do not sum per slot.** Bands overlap in wall-clock time (§1.2) — Gr12
at 08:00 runs alongside Gr10 from 07:30 — so a per-slot total misses
concurrent demand from the other band entirely. Demand must be evaluated
against the clock:

```
demand(t) = Σ rooms(p)   over papers p with start(p) ≤ t < start(p) + occupancy(p)
peak(day) = max over t of demand(t)
```

Compute it with a **sweep line** over interval endpoints — sort the day's
paper starts and ends, walk them accumulating `rooms(p)`, and take the
running maximum. That is `O(P log P)` per day, needs no time grid at all,
and sidesteps the misaligned 30-minute band offset cleanly. Do not build a
minute-by-minute array to do this; the sweep is simpler and exact.

Two constraints follow:

- **Rooms — a cheap guard.** `rooms(slot) ≤ V`. Rarely binding, since `V`
  seats the whole school (§1). The one way it bites is **ceiling waste**:
  twenty papers of 5 candidates need 20 rooms but only 100 seats' worth of
  students. A sitting crowded with small subjects can therefore exceed `V`
  while looking half-empty by headcount. Keep the check; expect it to pass.
- **Invigilators — the real constraint.** `rooms(slot)` teachers must be
  free for the sitting's whole occupancy, costed as
  `rooms(p) × periods(p)` **invigilator-periods**. This is what actually
  limits how much can run concurrently, and it is what §6 must satisfy.

The same ceiling waste that threatens rooms costs invigilators one-for-one,
and there it is never slack: those 20 small papers need 20 teachers
standing in 20 rooms. **Consolidating small papers into shared sittings
does not help** — papers cannot share a venue — so the lever is placing
small papers where teachers are free, not packing them together.

This is the one place cohort size legitimately enters the objective, and it
is why cohort size must not *also* drive student stress (§0.1).

### 3.4 Invigilator supply — derived, not assumed

Supply is a **curve over the season and over each day**, not a constant.
A teacher is available at instant `t` only if they are not teaching then and
not over their marking cap (§3.2).

Teaching commitments come from `timetable.json`, but mapping them onto the
exam wall clock needs two things the JSON does not carry:

- **Bell times.** `timeslots` are labels only — `["A1",…,"H7","P1",…,"P4"]`
  (`raw/sources/timeedusuite-core.md:165`), with no clock times. TimeView
  already holds them: its reference times table gives start/end per period
  **per day type** — Normal / Assembly / Test / Long Reg
  (`raw/sources/timeedusuite-timeview.md:41`). So a day's bell schedule
  depends on its day type, and that must be an input.
- **Cycle-to-calendar mapping.** The teaching timetable is an 8-day cycle
  (8 blocks × 7 periods, `raw/sources/timeedusuite-core.md:53`), not a
  Mon–Fri week. Knowing which cycle-day a given calendar date is determines
  who is teaching. Without this mapping, availability cannot be computed at
  all.

**The junior-teaching effect.** While Gr8–9 are still in class (§1.2), any
teacher with a junior lesson in that period is unavailable. Derive this from
the timetable rather than special-casing "junior teachers" — staff are not
partitioned by grade, and a Maths teacher may hold both Gr9 and Gr12
classes.

> **An adverse selection worth naming.** The teachers most likely to be free
> during senior exams are those who teach only senior grades — who are
> precisely the people marking the senior papers. Teachers with junior loads
> are busy teaching. So the invigilation pool is biased *against* the staff
> with the lightest marking, which is the opposite of what §4.2 wants. This
> sharpens the §4.5 conflict rather than replacing it, and it is the main
> reason the two-phase split (§6) needs its feedback loop.

---

## §4 Objective function

### §4.0 Lexicographic, not weighted — and why

**Requirement: the schedule must be provably optimal.** That constrains the
whole design, and it is why the student and teacher objectives are ordered
rather than blended.

**The two stages are strictly lexicographic:**

1. Minimise the student objective `Z_stu`. Solve to **proven** optimality,
   per grade. Call the result `Z*_g`.
2. Add `Z_stu(g) = Z*_g` as a **hard constraint** for every grade, then
   minimise the teacher objective `Z_tch` subject to it.

Stage 2 therefore searches only the set of student-optimal schedules. It
can never trade a student's fairness for a teacher's convenience, which is
exactly the stated requirement: teacher needs *filtered in afterwards
without changing the student score*.

**This does not contradict the weighted-sum advice elsewhere in this spec.**
Within the student term, the window sizes w ∈ {2,3,4,5} are combined as a
weighted sum (§4.1) — a lexicographic cascade *there* would stall, since
each level is satisfiable many ways and gives the next nothing to work
with. Between student and teacher, lexicographic is right, because the
priority is genuine and absolute. Different question, different answer.

**Why ties are the point.** Stage 2's freedom is precisely the size of the
student-optimal set. Coarse `SF` (1–3) produces many ties, and every tie is
a schedule stage 2 may choose between at zero cost to students. This is a
deliberate design choice, not a modelling accident — see §1.3.

### §4.0.1 Keep the student objective in exact integers

`Z_stu = Z*` is a hard equality constraint, so it must be **exact**. Any
floating-point drift makes stage 2 either infeasible or silently permissive.

- `SF` is a small integer; keep every weight (`λ_w`, thresholds) integral.
- The fair share `Λ_s × w / D_s` is rational — **do not divide.** Multiply
  through instead, comparing `α_den · W_{s,w}(d) · D_s` against
  `α_num · Λ_s · w`, with `α` given as a ratio of integers. All comparisons
  and penalties then stay in integer arithmetic.
- `φ(x) = max(0, x)^p` over bounded integers is integral, and small enough
  to tabulate rather than compute.

No floating point anywhere in `Z_stu`.

### §4.0.2 What "optimal" does and does not mean

The proof available here is: **`Z_stu` is minimal over all feasible
schedules, and `Z_tch` is minimal among those achieving it.** That is a
real, checkable guarantee and it is what a solver's optimality status
certifies.

It is *not* a proof that the schedule is the fairest possible in any
absolute sense — only that it is optimal **with respect to this objective
function**. If `SF`, `λ_w` or the fair-share model mis-describe real
stress, the result is provably optimal for the wrong measure. Worth being
precise about when presenting the guarantee: the arithmetic is proven, the
modelling is a judgement. Calibrating the weights (below) is therefore not
tuning for taste — it is the part of the claim that is not self-certifying.


One weighted sum with named, tunable coefficients. **Not** a lexicographic
cascade — "perfect the 2-day balance, then the 3-day" stalls immediately,
because the first level is satisfiable in many ways and gives the later
levels nothing to work with.

### 4.1 Student fairness — the primary term

Each student is measured against **their own fair share**, not a global
threshold.

Let `Λ_s = Σ SF(p)` over all papers student `s` writes, and `D_s` = the
number of exam days **in that student's own band window** (§1.2). Student
`s`'s fair share of any `w`-day window is:

```
fair_s(w) = Λ_s × w / D_s
```

**`D_s` is per student, not global — this matters.** The bands' windows are
nested: seniors write over more weeks, juniors are compressed into fewer,
and all end together. A Gr9 student with 8 papers in two weeks faces a
genuinely denser schedule than a Gr12 with 10 papers over four, and a global
`D` would score the junior as comfortable while they are in fact the most
squeezed student in the school. Using each student's own window makes the
two directly comparable — which is the entire point of normalising.

Let `W_{s,w}(d)` be the `w`-day sliding-window sum of `L_s` starting at day
`d`. The excess over fair share, with slack multiplier `α ≥ 1`:

```
excess_{s,w}(d) = W_{s,w}(d) − α · fair_s(w)
```

And the term:

```
Student = Σ_s Σ_{w ∈ {2,3,4,5}}  λ_w  Σ_d  max(0, excess_{s,w}(d))^p
```

**Why per-student normalisation matters.** Judged against one global
threshold, a 9-subject student is permanently over it and a 6-subject
student never near it — so the optimiser spends its entire budget on the
heavy-load students and does nothing for anyone else. Normalising makes
every student's excess directly comparable, which is what "fair for as many
students as possible" actually requires.

**`p` is the fairness dial:**

| `p` | Behaviour |
|---|---|
| 1 | Utilitarian. Indifferent between one student 20 over and five students 4 over. Will sacrifice a few badly. |
| **2** | **Default.** Strongly prefers spreading excess across students. |
| →∞ | Approaches minimax — protects the worst-off at the average's expense. |

This replaces the brief's explicit "if one student is over threshold, spread
it over others" mechanic. A convex penalty produces that behaviour for free.

**On "as many students as possible" — and a note that changed.** An earlier
draft argued the literal objective (a *count* of students below their fair
share) was unoptimisable: piecewise-constant, no gradient, invisible to
local search. **That objection dies with CP-SAT.** A count is a sum of
boolean indicators, which an exact solver handles natively — so the literal
reading of the goal is directly available:

```
maximise  Σ_s  headcount(s) · [ every window of s is within fair share ]
```

Three options, then, and the choice is a real one:

| Form | Behaviour |
|---|---|
| **Convex excess** (above) | Discriminates among failing students — a student 2 over is better than one 10 over. **Default.** |
| **Count** | Matches the stated goal literally, but is indifferent to *how badly* the failures fail. |
| **Count, then excess** | Lexicographic: maximise satisfied students, then minimise the remaining excess. Most faithful; costs the most ties. |

Recommend starting with **convex excess** — it captures most of the count's
intent while still caring about severity — and trying the third only if
real output shows it sacrificing a few students badly.

> **Every refinement of the student objective eats stage-B headroom.** A
> finer or more layered `Z_stu` orders more schedules strictly, shrinking
> the optimal set that teacher needs get to move within (§4.0). This is the
> same trade-off as coarse `SF` (§1.3), and it applies to every decision in
> this section: prefer the objective that is *just* discriminating enough.

**Optional refinement — CVaR.** Apply the term to only the **worst 10% of
students**, so effort is not spent on students already comfortably fine.
Worth trying once the basic version works; do not start here.

**Note on "balance in totality":** each student's total load `Λ_s` is fixed
by their subject choices regardless of placement, so total balance is a
constant and cannot be optimised. It is not useless, though — it is exactly
what defines each student's fair baseline above.

`λ_w` seeds from TimePyBling's proven weights: 3/2/1 for w=2/3/4, plus a
w=5 weight to be determined.

### 4.2 Teacher marking term

Convex penalty on `B_t(d)` peaks, same shape as §4.1, plus a fairness term
on total invigilation counts across teachers.

Unlike student load, teacher totals are **not** fixed — invigilation is
assignable, and marking distribution depends on how cohorts split across
markers. So totality is genuinely optimisable on the teacher side.

### 4.3 Second-session usage

A penalty per session-2 sitting used, plus a fatigue multiplier on student
stress for papers written in it (an afternoon paper is harder than the same
paper in the morning).

**Deliberately a cost, not a gate.** The brief specified "only use session 2
after session 1 is full". A hard gate is brittle — "full" is undefined (all
grades placed? venues exhausted?) — and it creates ordering artefacts where
the schedule depends on the sequence papers were considered in. A penalty
achieves the same preference and degrades gracefully when second sittings
are genuinely needed.

**The penalty is not arbitrary — derive it from the clock.** Per §2.4,
session 2 starts after session 1's longest paper finishes, so a long first
sitting pushes the second one late into the afternoon. Scale the fatigue
multiplier by that start time rather than picking a constant: a session 2
starting at 11:00 is barely a penalty; one starting at 15:00 is a real one.
This also creates the right secondary pressure — it discourages pairing a
very long session-1 paper with a session-2 sitting at all.

Note this makes §4.3 the one term where paper *duration*, not just `SF`,
enters student stress.

### 4.4 Front-loading heavy marking

```
Frontload = Σ_p  M_p × dayIndex(p)
```

Weighted, and **explicitly in tension with §4.1** — the heaviest-marking
papers are the biggest cohorts, which are the universal subjects nearly
every student writes. Pushing them early spikes student load in week 1.
Expose the weight so the trade-off is tunable rather than baked in.

### 4.5 Departmental co-freedom

Reward common free sessions across all of `markers(p)` in the window after
paper `p` — big subjects mark together, so the department wants a shared
free block.

**Explicitly in tension with invigilation cover** (§6): a free Maths
department means someone else invigilates, and that someone is
systematically a small-subject teacher. The invigilation fairness term in
§4.2 is what keeps this honest.

---

## §5 Solving — exact, in two lexicographic stages

Proving optimality rules out local search as the primary method. Hill
climbing and annealing return a local optimum **with no bound**: they can
say "this is the best I found", never "this is the best that exists". If
the schedule must be provably optimal, the solver must produce a matching
bound — which means an exact method.

**Use CP-SAT.** OR-Tools is the natural fit: it proves optimality on
discrete scheduling models, and the sibling [[timemath-project]] already
runs Python + OR-Tools CP-SAT
(`wiki/entities/timemath-project.md:13`) with a bounds ladder for exactly
this kind of argument (`wiki/concepts/bounds-ladder.md`). The institutional
experience is already in the building.

### Stage A — student optimum, per grade, proven

The student objective is separable by grade (§0), so each grade is its own
model. Solve each to proven optimality and record `Z*_g`.

Grade-level decomposition is what makes exactness realistic: a whole-school
model would be large and coupled; one grade's papers over one band's window
is a model CP-SAT can plausibly close.

**Collapse students into subject-set classes first.** Students with
identical subject sets have identical loads under every schedule, so they
need not appear separately: group them, and weight each class by its
headcount. A grade of 200 students typically holds far fewer distinct
subject combinations, and the model shrinks by that ratio with **no loss of
exactness** — the objective is unchanged, just factored.

This is the same insight as [[basket-sdr]] in the sibling project, where
what matters is the set of *actual* subject combinations rather than all
pairwise possibilities. Reuse the concept; the wiki page has the reasoning.

### Stage B — teacher optimum within the student optimum

Fix `Z_stu(g) = Z*_g` for every grade as a hard constraint, then minimise
the teacher objective across all grades jointly (marking balance,
invigilation fairness, front-loading, departmental co-freedom).

Stage B is where the grades finally meet: it carries the cross-grade hard
constraints — the invigilator sweep (§3.3) and `rooms ≤ V` — and the
teacher terms that are the only genuine coupling.

**Note the ordering risk.** Stage A optimises each grade in isolation, so
the per-grade optima may not be **jointly staffable**. Two remedies, in
preference order:

1. Carry the cross-grade invigilator constraint in Stage A too, solving the
   grades **simultaneously but with only the student objective**. Keeps the
   guarantee exact and is the correct formulation; costs model size.
2. If that will not close, solve grades independently, then let Stage B
   relax `Z_stu(g) = Z*_g` to `≤ Z*_g + ε_g` where infeasibility demands
   it — reporting `ε_g` explicitly, since a non-zero `ε` is precisely the
   amount by which the "no student cost" guarantee was broken.

Prefer (1). Reach for (2) only with the numbers in hand, and never
silently.

### If CP-SAT cannot close the gap

At full scale the model may not prove optimality in acceptable time. Do not
misreport that as a proof. The graceful degradation:

- Report the **gap**: incumbent value vs. best bound, i.e. "within `x`% of
  optimal", which is still a far stronger claim than local search offers.
- Feed CP-SAT a good incumbent as a hint to tighten the gap. This is the
  sanctioned use of hints per `wiki/concepts/warm-start-rejected.md` — from
  the solver's own heuristic stage on this run's data, never from a prior
  year's schedule, which that page rejects on evidence.
- Strengthen the bound analytically rather than searching harder. The
  bounds-ladder approach from timemath applies: derive a floor for
  `Z_stu` from structure (each student's own load is fixed, so their
  windows cannot all be under-full), and a solution meeting the floor is
  optimal with no search at all.

### Where the heuristic machinery still earns its place

The local-search design from earlier drafts is **not wasted** — it is
demoted from primary method to supporting role:

- producing incumbents to hint CP-SAT and to bound the gap;
- the fallback if exactness proves unreachable at this school's scale;
- fast interactive what-if once a schedule exists.

Its incremental-delta requirement stands for those uses: maintain
per-student per-day load arrays and per-teacher backlog arrays, update only
affected entries. Full re-evaluation per move is what made the Python
version unusable.

**Weeks remain a calendar structure, never a processing order** — under an
exact method this is automatic, since the solver considers the horizon as a
whole. It only needs restating if the heuristic path is taken.

---

## §6 Invigilation — second phase

Placement decides *when* papers happen. Invigilation decides *who watches*.
These are separable, and separating them is what makes the problem
tractable.

**During placement:** a cheap feasibility surrogate only — free-teacher
headcount for the sitting ≥ `rooms(slot)`. No assignment, no matching.

**After placement:** min-cost bipartite matching (or min-cost flow) of
teachers to sessions.

- Cost of assigning teacher `t` to a session on day `d` = their marking
  backlog `B_t(d)` — busy markers get fewer duties.
- Hard exclusions: own-subject invigilation; backlog over cap.
- Demand per slot = `rooms(slot)` from §3.3.
- Duty is costed in **invigilator-periods** (from occupancy, §1.3), not
  sittings — a 6-period paper is three times the commitment of a 2-period
  one, and fairness across teachers must be measured in periods or it will
  quietly favour whoever draws the short papers.

**This phase is where the design is most likely to fail feasibility.** With
rooms ample and invigilators scarce (§1), the matching is the step that
discovers a placement cannot be staffed. Two guards worth building in from
the start: report the tightest slot (demand ÷ supply) so the failure is
legible, and make the surrogate during placement conservative enough that
the matching rarely fails outright — a placement that scores well and then
cannot be staffed wastes a whole optimisation run.

This is solvable exactly and quickly for a fixed timetable, which is why it
must not live inside the local-search inner loop.

All inputs are constants (`VENUE_PAX`, `R`, `PERIOD`, `V`) or derive from
the timetable, so this phase can be built as soon as the marking model
(§3.2) is in place.

**Feedback:** the resulting imbalance re-enters the placement objective
(§4.2) for a subsequent round. Two or three outer iterations should
converge; more suggests the surrogate is too weak.

---

## Requirements traceability

Every requirement from the original brief:

| # | Requirement | Where |
|---|---|---|
| 1 | Import `timetable.json` v3.1 + exam xlsx | §1 — xlsx also supplies paper duration |
| 2 | Group students and teachers per subject | §2.2 |
| 3 | Sessions week by week (Mon–Fri) | §1.2 per-band sittings and nested windows; §2.4 calendar; §5 — weeks are structure, not processing order |
| 4 | Constants file, SF per subject as multiplier | §1 — kept at 1–3; ties are load-bearing (§4.0) |
| 5 | Score = students × SF | §3 — **split into three quantities**, see §0.1 |
| 6 | Place minimising score for the session | §3.3 venue/invigilator demand, §4 objective |
| 7 | Two conflicting heuristics: marking vs writing | §3.1, §3.2, §4.1, §4.2 |
| 8 | Balance 2/3/4/5 consecutive days, then week, then totality | §4.1 — totality is constant per student, becomes the fair baseline |
| 9 | Threshold, spread excess over other students | §4.1 — **convex penalty gives this for free** |
| 10 | Invigilation vs marking; cannot mark and invigilate | §3.2 backlog model, §6 |
| 11 | Big subjects mark together; department free together | §4.5 |
| 12 | Heaviest marking earlier | §4.4 |
| 13 | Second session, only after firsts are full | §4.3 fatigue cost, not a gate; §1.2 adds a hard fit check (≤ 3 h 45 in session 1) |
| 14 | Link two exams same day (session 1 + 2) | §2.4 link groups; §5 atomic moves |
