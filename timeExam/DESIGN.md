# TimeExam — design specification

Draft 1, 2026-09-01. Supersedes the TimePyBling scoring model.

## §0 Goal, and what changed from the brief

**Goal:** produce an exam schedule that is *as fair as possible for as many
students as possible*, while keeping teacher marking load survivable and
invigilation covered.

That goal is a **coverage** objective, not a sum-minimisation one, and it
drives most of the design decisions below. Five changes from the original
brief, each with its reason:

1. **`students × SF` is split into three separate quantities.** Cohort size
   drives marking effort and seating; it has no bearing on how stressed an
   individual student is. Collapsed into one number, the "two conflicting
   heuristics" are actually one quantity and a cohort-scaled multiple of
   it — they cannot genuinely trade off against each other. See §3.
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
| Papers | xlsx | Papers written, per grade per subject |
| Constants | `constants/subjects.*` | `SF`, `MF`, duration per subject |
| Seating | **not yet sourced** | See below |

**Schema version caution.** TimeWiki documents `timetable.json` at v2.1/v3.0
(`wiki/concepts/json-timetable-schema.md`). v3.1 is newer than anything
ingested here, so the delta is unknown. Confirm the live schema before
implementing the reader; do not assume the documented shape is current.

**Missing input.** Exam venue and seating capacity is not in
`timetable.json`. Lessons carry a *teaching* venue, which is not the same
quantity as exam seating — exam venues are typically halls, with a seat
count and an invigilator-per-N-candidates ratio. This needs a third input
file that does not exist yet. Until it does, session capacity (§3.3) cannot
be checked and must be stubbed.

### Constants file

Per subject: `SF`, `MF`, `duration`.

**`SF` — student stress factor, integer 1–5.** The brief specified 1–3;
widened because with ~7 subjects per student and only three levels, large
numbers of students tie on score, and ties give local search nothing to
discriminate between. Widening costs nothing.

`SF` is a per-subject constant, so a student's total stress is fully
determined by their subject choices. What `SF` deliberately does *not*
capture is that two SF-5 subjects back to back are worse than their sum —
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
machinery for this already exists for the AM/PM linkage case (§2.4), so the
cost is modelling verbosity, not new code. If cross-grade papers turn out to
be common rather than exceptional, revisit the key.

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
- `Slot = (day, session)`, `session ∈ {AM, PM}`.
- Week index derives from day index; weeks are a *structure*, never a
  processing order (§5).

**Link group** — a set of papers that must be placed together with a fixed
internal arrangement. Covers three cases with one mechanism:

- two papers on the same day, AM + PM (the brief's explicit requirement)
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

### 3.3 Session capacity — per slot

```
C(slot) = Σ |cohort(p)|   for p in slot
```

Checked against seats and invigilator supply. **Blocked on the missing
seating input** (§1) — stub until that file exists.

---

## §4 Objective function

One weighted sum with named, tunable coefficients. **Not** a lexicographic
cascade — "perfect the 2-day balance, then the 3-day" stalls immediately,
because the first level is satisfiable in many ways and gives the later
levels nothing to work with.

### 4.1 Student fairness — the primary term

Each student is measured against **their own fair share**, not a global
threshold.

Let `Λ_s = Σ SF(p)` over all papers student `s` writes, and `D` = number of
exam days. Student `s`'s fair share of any `w`-day window is:

```
fair_s(w) = Λ_s × w / D
```

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

**On "as many students as possible":** the literal objective is a *count* of
students below their fair share, which is unoptimisable — piecewise
constant, zero gradient almost everywhere, local search cannot navigate it.
The convex-excess form above is its usable surrogate: satisfied students
contribute exactly zero, so minimising it drives students under the line.

**Optional refinement — CVaR.** To target the goal more directly, apply the
term to only the **worst 10% of students** rather than all of them. Still
convex, still differentiable enough for local search, but it stops spending
effort on students who are already comfortably fine. Worth trying once the
basic version works; do not start here.

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

### 4.3 PM session usage

A penalty per PM session used, plus a PM multiplier on student stress
(fatigue is real — an afternoon paper is harder than the same paper in the
morning).

**Deliberately a cost, not a gate.** The brief specified "only use session 2
after session 1 is full". A hard gate is brittle — "full" is undefined
(venue capacity? all grades placed?) — and it creates ordering artefacts
where the schedule depends on the sequence papers were considered in. A
penalty achieves the same preference and degrades gracefully when PM
sessions genuinely are needed.

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

## §5 Search

**Global over the whole horizon.** Weeks are a calendar structure and a
balance level — never a processing order. Placing week 1, then week 2,
cannot revisit a week-1 decision that only looks wrong once week 3 lands.
TimePyBling's hill-climb was already global; do not regress to sequential
week filling.

**Construction:** greedy seed, papers in descending `|cohort| × MF`.

**Improvement:** local search (hill-climb, or simulated annealing if the
landscape proves rugged). Moves:

- swap two papers' slots
- move a paper to a free slot
- move a link group atomically (§2.4)

**Incremental delta evaluation is the load-bearing element of this design.**
Maintain per-student per-day load arrays and per-teacher backlog arrays;
a move updates only the affected entries. Full re-evaluation per move is
what made the Python version unusable, and no amount of C++ rescues an
`O(students × days)` inner loop.

---

## §6 Invigilation — stage 2

Placement decides *when* papers happen. Invigilation decides *who watches*.
These are separable, and separating them is what makes the problem
tractable.

**During placement:** a cheap feasibility surrogate only — free-teacher
headcount per session ≥ invigilators required. No assignment, no matching.

**After placement:** min-cost bipartite matching (or min-cost flow) of
teachers to sessions.

- Cost of assigning teacher `t` to a session on day `d` = their marking
  backlog `B_t(d)` — busy markers get fewer duties.
- Hard exclusions: own-subject invigilation; backlog over cap.
- Capacities from the seating file (§1).

This is solvable exactly and quickly for a fixed timetable, which is why it
must not live inside the local-search inner loop.

**Feedback:** the resulting imbalance re-enters the placement objective
(§4.2) for a subsequent round. Two or three outer iterations should
converge; more suggests the surrogate is too weak.

---

## Requirements traceability

Every requirement from the original brief:

| # | Requirement | Where |
|---|---|---|
| 1 | Import `timetable.json` v3.1 + exam xlsx | §1 |
| 2 | Group students and teachers per subject | §2.2 |
| 3 | Sessions week by week (Mon–Fri) | §2.4 calendar; §5 — weeks are structure, not processing order |
| 4 | Constants file, SF per subject as multiplier | §1; widened to 1–5 |
| 5 | Score = students × SF | §3 — **split into three quantities**, see §0.1 |
| 6 | Place minimising score for the session | §3.3, §4 |
| 7 | Two conflicting heuristics: marking vs writing | §3.1, §3.2, §4.1, §4.2 |
| 8 | Balance 2/3/4/5 consecutive days, then week, then totality | §4.1 — totality is constant per student, becomes the fair baseline |
| 9 | Threshold, spread excess over other students | §4.1 — **convex penalty gives this for free** |
| 10 | Invigilation vs marking; cannot mark and invigilate | §3.2 backlog model, §6 |
| 11 | Big subjects mark together; department free together | §4.5 |
| 12 | Heaviest marking earlier | §4.4 |
| 13 | Second session, only after firsts are full | §4.3 — **cost, not gate**, with reason |
| 14 | Link two exams same day (session 1 + 2) | §2.4 link groups; §5 atomic moves |
