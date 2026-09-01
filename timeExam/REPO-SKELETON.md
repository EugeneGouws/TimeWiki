# TimeExam — proposed repository layout

For the new repo, following the suite's layered convention (pure core →
io → app → ui), which the archived notes call out as the lesson that
"carried to C++ ports".

## Language recommendation

**C++ for the solver, headless and CLI-first.**

Reasoning: performance is the *known* blocker — it is the documented reason
TimePyBling was abandoned, and the inner loop (§5 of `DESIGN.md`) is exactly
the kind of tight bitset-and-array work where C++ wins by two orders of
magnitude. The suite architecture already anticipates TimeExam as a C++/Qt
desktop tool linking `timeedu_core`
(`wiki/entities/timeedusuite-core-library.md`).

Two qualifications:

- **CLI-first, UI later.** The solver must be runnable and testable without
  Qt. The whole objective function is verifiable from a terminal against a
  fixture; a UI dependency in the core would make that painful.
- **Don't rebuild the Python prototype first.** The suite's own
  prototype-then-port pattern
  (`wiki/concepts/python-prototype-to-cpp-pattern.md`) has now cost a full
  rewrite three times. The algorithm here is specified, not exploratory. A
  small Python *harness* for weight tuning and plotting is worth having —
  it reads the solver's JSON output, it is not a second implementation.

**Open:** whether to link `timeedu_core` from the start. It supplies the
domain model and JSON I/O, which is real leverage, but it also couples
TimeExam's release to the core's versioning. See `OPEN-QUESTIONS.md`.

## Layout

```
timeexam/
├── CLAUDE.md               project guide, layer rules, build state
├── README.md
├── docs/
│   ├── DESIGN.md           the spec (from this staging dir)
│   ├── PORT-NOTES.md
│   └── OPEN-QUESTIONS.md
│
├── constants/
│   ├── subjects.json       SF, MF, duration per subject
│   └── weights.json        λ_w, α, p, K, term weights — tunable, not code
│
├── src/
│   ├── model/              pure domain, no I/O, no Qt
│   │   ├── paper.h/.cpp        Paper, identity, cohort bitset
│   │   ├── calendar.h/.cpp     days, slots, weeks, horizon
│   │   ├── linkgroup.h/.cpp    composite placeable units
│   │   └── schedule.h/.cpp     the assignment being optimised
│   │
│   ├── io/
│   │   ├── timetable_reader   timetable.json v3.1 → model
│   │   ├── paper_reader       exam xlsx → papers
│   │   ├── constants_reader   constants/*.json
│   │   ├── seating_reader     venue capacity (blocked — see OPEN-QUESTIONS)
│   │   └── writer             schedule → JSON + xlsx export
│   │
│   ├── score/              pure, no I/O — the heart of the thing
│   │   ├── student_load       L_s(d), windows, fair baselines
│   │   ├── marking_load       M(t,p), backlog B_t(d)
│   │   ├── capacity           session seat/invigilator demand
│   │   ├── objective          weighted sum of all terms
│   │   └── deltas             incremental update on a move
│   │
│   ├── search/
│   │   ├── construct          greedy seed
│   │   ├── moves              swap, relocate, move-link-group
│   │   └── optimise           hill-climb / annealing driver
│   │
│   ├── invigilation/
│   │   ├── matcher            min-cost bipartite assignment
│   │   └── feedback           imbalance → placement penalty
│   │
│   ├── verify/                independent checker — see below
│   └── cli/                   entry point, run modes
│
├── tests/
│   ├── fixtures/              small hand-verifiable schools
│   ├── score/                 objective correctness
│   ├── deltas/                delta == full recompute (critical)
│   └── verify/
│
└── tools/
    └── tuning/                Python harness: read output, plot, sweep weights
```

## Two structural notes

**`score/deltas` needs its own test suite.** Incremental evaluation is the
design's performance bet (`DESIGN.md` §5), and delta bugs are the classic
silent killer — the optimiser happily descends a corrupted objective and
produces a confidently wrong schedule. Every move type needs a test
asserting `delta_update(state, move) == full_recompute(apply(state, move))`
over randomised inputs. Treat a failure here as a correctness bug, not a
tuning issue.

**`verify/` must share no code with `score/`.** An independent checker that
re-derives clashes, capacity violations and load statistics from the output
schedule alone. This is a suite-wide principle — `ClashDetector` as an
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
2. `score/student_load` + `objective` (student term only) — scoring a
   *hand-made* schedule. Verifiable by hand on a fixture.
3. `search/` with the student term alone — first real output. Judge whether
   the fairness model actually produces schedules that look fair before
   adding terms.
4. `score/marking_load` + the teacher terms.
5. `invigilation/` + feedback loop.
6. Export, then UI.

Steps 1–3 are the risky part: if the fairness model is wrong, everything
after it is wasted. Get a schedule a human agrees is fair before building
out.
