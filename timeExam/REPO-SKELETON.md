# TimeExam — proposed repository layout

For the new repo, following the suite's layered convention (pure core →
io → app → ui), which the archived notes call out as the lesson that
"carried to C++ ports".

## Language recommendation

**Python + OR-Tools CP-SAT for the solver, headless and CLI-first.**

This reverses an earlier draft of this file, which argued for C++. The
optimality requirement (`DESIGN.md` §5) changed the answer: with CP-SAT
doing the search, the host language builds the model and reads results —
work that is `O(papers)` and trivial — while everything expensive runs
inside OR-Tools' C++ core. Writing the model in C++ buys almost nothing.

**The TimePyBling precedent does not apply here.** Its blocker was a
hand-rolled Python hill-climb, where the interpreter *was* the inner loop.
That is a genuinely different situation from calling into a compiled
solver, and the lesson does not transfer.

The sibling [[timemath-project]] already runs Python + OR-Tools CP-SAT
(`wiki/entities/timemath-project.md:13`), so this also shares tooling and
experience with the project most like it.

**Reconsider C++ only if** the exact approach cannot close the gap at this
school's scale and the fallback heuristic (`DESIGN.md` §5) becomes the
primary method. Then the inner loop is hand-written again and the original
argument returns in full. Structure the repo so that swap is possible:
keep `model/` and `score/` free of solver-specific types.

**CLI-first, UI later.** The solver must be runnable and testable without a
GUI — the objective is verifiable from a terminal against a fixture, and a
UI dependency in the core would make that painful.

**Note on `timeedu_core`:** the suite architecture anticipated TimeExam as a
C++/Qt tool linking the shared core
(`wiki/entities/timeedusuite-core-library.md`). A Python solver breaks that
assumption. The JSON contract is the real integration point and is
language-neutral, so this is a deviation worth recording rather than a
problem — but it should be a conscious decision, not a drift.
See `OPEN-QUESTIONS.md`.

## Layout

```
timeexam/
├── CLAUDE.md               project guide, layer rules, build state
├── README.md
├── docs/
│   ├── DESIGN.md           the spec (from this staging dir)
│   ├── LITERATURE.md       positioning, citations, terminology
│   ├── PORT-NOTES.md
│   └── OPEN-QUESTIONS.md
│
├── constants/
│   ├── subjects.json       SF (1-3), MF per subject
│   ├── weights.json        λ_w, α as int ratio, p, K — all integral (§4.0.1)
│   └── venues.json         V, VENUE_PAX=25, R=25, PERIOD=40, OVERHEAD=4/3
│
├── src/timeexam/
│   ├── model/              pure domain — no I/O, no solver types
│   │   ├── paper.py            Paper, identity, cohort
│   │   ├── calendar.py         days, bands, per-band sittings, weeks
│   │   ├── bands.py            band windows, sitting clock times
│   │   ├── baskets.py          collapse students to subject-set classes (§5)
│   │   └── schedule.py         the assignment being solved
│   │
│   ├── io/
│   │   ├── timetable_reader.py timetable.json v3.1 → model
│   │   ├── paper_reader.py     exam xlsx → papers (+ duration)
│   │   ├── constants_reader.py constants/*.json
│   │   ├── bells_reader.py     period clock times per day type (TimeView)
│   │   └── writer.py           schedule → JSON + xlsx export
│   │
│   ├── score/              pure, integer-only, no solver dependency
│   │   ├── student.py          L_s(d), windows, per-band fair baselines
│   │   ├── marking.py          M(t,p), backlog B_t(d)
│   │   ├── rooms.py            demand sweep-line over the clock (§3.3)
│   │   ├── supply.py           teacher availability curve (§3.4)
│   │   └── objective.py        Z_stu and Z_tch, evaluated on a schedule
│   │
│   ├── solve/              CP-SAT — the only solver-aware package
│   │   ├── stage_a.py          student model, per grade, to optimality
│   │   ├── stage_b.py          teacher model, Z_stu pinned to Z*
│   │   ├── bounds.py           analytic floors (bounds-ladder style)
│   │   └── report.py           status, objective value, gap
│   │
│   ├── invigilation/
│   │   ├── matcher.py          min-cost bipartite assignment
│   │   └── feedback.py         imbalance → placement penalty
│   │
│   ├── heuristic/          CONTINGENCY only — build if step 4 fails (§5)
│   │   ├── construct.py        greedy seed
│   │   ├── moves.py            swap, relocate, move-link-group
│   │   └── deltas.py           incremental update on a move
│   │
│   ├── verify/             independent checker + parent report (§7)
│   │   ├── checker.py          re-derive clashes, caps, staffing
│   │   ├── distribution.py     percentiles, CVaR-10%, Jain, Gini, per grade
│   │   └── student_report.py   per-learner report + why-not-better
│   └── cli.py              entry point, run modes
│
├── tests/
│   ├── fixtures/           small hand-verifiable schools
│   ├── score/              objective correctness, integer invariants
│   ├── solve/              optimality on fixtures with known answers
│   ├── deltas/             delta == full recompute (if heuristic is used)
│   └── verify/
│
└── tools/
    └── tuning/             weight calibration: sweep, plot, compare

```

**`score/` must not import from `solve/`.** The objective has to be
evaluable on any schedule — one CP-SAT produced, one the heuristic
produced, one a human typed in — and it is what `verify/` checks against.
Entangling it with solver variables makes all three impossible.

## Two structural notes

**`heuristic/deltas` needs its own test suite** *(only if the heuristic path
is taken).* Incremental evaluation is that path's
performance bet (`DESIGN.md` §5), and delta bugs are the classic
silent killer — the optimiser happily descends a corrupted objective and
produces a confidently wrong schedule. Every move type needs a test
asserting `delta_update(state, move) == full_recompute(apply(state, move))`
over randomised inputs. Treat a failure here as a correctness bug, not a
tuning issue.

**`verify/` must share no code with `score/`.** An independent checker that
re-derives clashes, invigilation shortfalls and load statistics from the
output schedule alone. It also emits the **per-student fairness report**
(`DESIGN.md` §7) — the artefact that answers a questioning parent. That it
is an independent re-derivation is precisely what makes the report
credible. This is a suite-wide principle — `ClashDetector` as an
independent verifier is the one concept the timemath project imported across
its firewall from TimeEduSuite
(`wiki/concepts/independent-verification.md`).

It matters more than usual here: TimePyBling's 5-day spacing enforcement was
never verified (`PORT-NOTES.md`), so unlike TimeVerify → TimeEditor there is
**no trustworthy oracle** to check the new implementation against. The
verifier is the only safety net.

## Suggested build order

1. `model/` + `io/timetable_reader` — get real cohorts out of a real
   `timetable.json` and print them. Proves the schema and the cohort
   derivation before anything depends on it.
2. **`score/rooms` + `score/supply` as a standalone diagnostic** — per
   paper, `ceil(|cohort|/25)` rooms and `rooms × periods`
   invigilator-periods; then total demand against teacher-period supply.
   A few hours' work, and it answers what everything else depends on:
   *how much slack is there in teacher time?* Rooms are ample
   (`DESIGN.md` §1), so this is a staffing calculation, not a space one.
   **Report it per phase** — weeks 1–2 (seniors only, junior teachers tied
   up) and weeks 3–6 (both bands writing, full pool) are different
   problems, and averaging hides the binding one (§1.2).
3. `model/baskets` + `score/student` — collapse students to subject-set
   classes, then score a *hand-made* schedule. Verify by hand on a fixture,
   and assert the integer invariants (§4.0.1) in tests from the start.
4. **`solve/stage_a` on one grade.** Confirm CP-SAT closes a single grade
   to proven optimality. The sample space is small, so this is expected to
   pass quickly — it is a checkpoint, not a research question. Run it
   anyway: it is cheap, and everything downstream assumes the answer.
5. `solve/bounds` — analytic floors for `Z_stu`. Worth doing even if step 4
   succeeds: a matching floor turns a slow proof into an instant one, and
   it is the fallback claim if the gap will not close.
6. `score/marking` + `solve/stage_b` — teacher terms with `Z_stu` pinned.
7. `invigilation/` matcher + feedback.
8. `verify/` — clash re-derivation, the distributional report, and the
   per-student fairness report (`DESIGN.md` §7). **Not last-minute
   polish:** requirement 3 is a third of the problem, and the report shapes
   what the objective must be able to explain.
9. Export, then UI.

**Two cheap things worth doing alongside step 8**, because both answer
questions that will definitely be asked: the **price of fairness** (compare
against the `p=1` schedule) and the **sensitivity table** (perturb `SF` ±1,
vary window weights). Neither needs new machinery — both are re-runs with
different constants — and together they are the most disarming artefacts to
bring to a hostile meeting.

**`heuristic/` is contingency only** — build it if step 4 unexpectedly
fails, not before. Leave the directory out of the initial repo rather than
stubbing it.

**Step 2 is the real gate**: whether the objective has room to operate
depends on invigilator slack, and that is not knowable without the numbers.
Step 4 is a checkpoint on an expectation, not an open question.

**Calibrating the weights is a build step, not a polish step.** The
optimality claim is only as meaningful as the objective it is proven
against (`DESIGN.md` §4.0.2), so `tools/tuning` earns its place early —
by step 6 at the latest, since that is when the teacher terms need
relative weights that survive scrutiny.
