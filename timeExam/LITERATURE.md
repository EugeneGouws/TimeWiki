# TimeExam — literature positioning

Where this design sits against published exam-timetabling practice, and the
vocabulary that makes it read as standard rather than ad hoc.

**Verification caveat.** This came from a research pass in which the agent's
egress proxy blocked every academic host (arXiv, Springer, ScienceDirect,
MDPI, Semantic Scholar, ACM, IEEE, PATAT, Nottingham, UniTime, Wikipedia).
Only web search and `raw.githubusercontent.com` were reachable, so **no full
paper was read**. Claims are tagged:

| Tag | Meaning |
|---|---|
| **[V-FILE]** | The actual artefact was downloaded and read. Highest confidence. |
| **[V-SNIP]** | A search engine returned quoted text from the source; the full paper was not read. |
| **[REF-OK]** | Bibliographic details appeared in results, so the reference exists and is named correctly — contents not vouched for. |
| **[RECALL]** | From model training data, **unverified**. A lead to check, not a citation. |

**Before any of this goes into a document that will be scrutinised, check
the [V-SNIP] and [RECALL] items against the originals.** Nothing here is
invented, but that is not the same as verified.

---

## 1. The two canonical formulations

### Carter, Laporte & Lee (1996) — the "Toronto" benchmark

Carter, M. W., Laporte, G., & Lee, S. Y. (1996). "Examination Timetabling:
Algorithmic Strategies and Applications." *Journal of the Operational
Research Society*, 47(3), 373–383. **[V-SNIP]**

The field's default benchmark (13 uncapacitated real instances) and its
most-cited objective, the **proximity cost**:

```
cost = (1/|S|) · Σ_{i<j} N_ij · w(|p_i − p_j|),    w(s) = 2^(5−s) for s ∈ 1..5
```

giving w(1)=16, w(2)=8, w(3)=4, w(4)=2, w(5)=1, where `N_ij` is the number
of students sitting both exams and `|S|` is the total student count.
**[V-SNIP]**

**Three properties that matter to us:**

1. It is a **pairwise sum over student-pairs**, not a load-over-window
   measure. There is no notion of "how much is this student carrying over
   the next three days" — which is precisely what we measure.
2. Normalisation is by **cohort size**, a constant rescaling for
   cross-instance comparability. It is **not** per-student normalisation and
   does nothing to make students comparable to each other.
3. A student with more exams contributes roughly quadratically more pairs
   and is structurally penalised harder, with no correction.

### ITC-2007, Examination Track — the modern standard

McCollum, B., McMullan, P., Parkes, A. J., Burke, E. K., & Qu, R. "The
Second International Timetabling Competition: Examination Timetabling
Track." Tech. Report QUB/IEEE/Tech/ITC2007/Exam/v4.0/17, Queen's University
Belfast, 2007. **[V-SNIP]** · McCollum et al. (2010), *INFORMS Journal on
Computing* 22(1), 120–130. **[REF-OK]**

Seven soft constraints, **verified from the OptaPlanner reference
implementation and the actual competition instance files** rather than from
prose **[V-FILE]**: `twoExamsInARow`, `twoExamsInADay`, `periodSpread`,
`mixedDurations`, `frontLoad`, `periodPenalty`, `roomPenalty`.

**The single most useful finding for us** — the `[InstitutionalWeightings]`
block, read from the eight competition instances **[V-FILE]**:

| Instance | TWOINAROW | TWOINADAY | NONMIXEDDURATIONS |
|---|---|---|---|
| set1 | 7 | 5 | 10 |
| set2 | 15 | 5 | 25 |
| set5 | 40 | 15 | 0 |
| set8 | 150 | 0 | 25 |

> **The flagship benchmark of the field treats its student-facing weights as
> declared institutional policy, varying more than 20× between institutions
> — one sets TWOINADAY to zero, i.e. does not care at all.** The literature
> does not prescribe these numbers. It asks each institution to state them.

That is the defence of `SF` and the window weights (§6 below).

Two details routinely got wrong in secondary sources, both **[V-FILE]**: the
number after `PERIODSPREAD` is a spread *length* in periods, not a weight
(the penalty is hard-coded to 1); and `twoExamsInARow` checks day equality,
so an evening→next-morning adjacency is **not** counted.

**There is no post-2007 competition standard for exam timetabling.**
ITC-2019 is university *course* timetabling **[V-SNIP]**. Worth saying
plainly if challenged — we are not ignoring a newer standard.

---

## 2. Deployed systems worth imitating

- **Bucknell.** Chaplin, Gai, et al. "Final Exam Scheduling at Bucknell
  University." *INFORMS Journal on Applied Analytics* (2025); preprint
  arXiv:2509.11031. **[REF-OK / V-SNIP]** Built visualisation first, then IP
  models producing a **portfolio of candidate schedules** for the Registrar
  to choose among. Headline: students with a back-to-back exam fell from
  ~⅓ to ~10%. **[V-SNIP]**
- **Cornell.** Ye, Jovine, et al. *INFORMS JAA* 56(2), 159–177 (2025).
  **[REF-OK]** Integer programming; minimise conflicts and consecutive
  exams per student per day, front-load large courses. **[V-SNIP]**
- **UniTime** (widely deployed, open source). Optimises direct conflicts,
  **"more than two exams a day"** conflicts, back-to-back conflicts, and
  distance back-to-back. **[V-SNIP]**

Both recent deployments are exact/IP methods — so our CP-SAT choice is
current best practice, not eccentricity.

---

## 3. Fairness in exam timetabling specifically

A small literature, dominated by one group.

- **Muklason, A., Parkes, A. J., McCollum, B., & Özcan, E. (2013).**
  "Initial Results on Fairness in Examination Timetabling." MISTA 2013.
  **[V-SNIP]**
- **Muklason, A., Parkes, A. J., Özcan, E., McCollum, B., & McMullan, P.
  (2017).** "Fairness in examination timetabling: Student preferences and
  extended formulations." *Applied Soft Computing*, 55, 302–318.
  **[V-SNIP]** · PhD thesis, Nottingham (2017) **[REF-OK]**

They name our exact problem:

> "Standard examination timetabling formulations concentrate on minimising
> the average penalty per student. However, this model can lead to
> unfairness, in that a small but still significant percentage of students
> may receive much higher than average penalties." **[V-SNIP]**

They survey students and find they care about fairness **relative to their
immediate cohort**, not the whole institution — which is a direct empirical
argument for our per-grade framing, and also for the cross-grade reporting
fix (`DESIGN.md` §4.1). They use a **multi-stage** approach (feasible →
quality → fairness), so lexicographic staging is established practice in
this exact domain. Measures used: max-min fairness and Jain's index.

- **Mühlenthaler, M., & Wanka, R. (2016).** "Fairness in academic course
  timetabling." *Annals of OR*, 239(1), 171–188. **[V-SNIP]** Course rather
  than exam, but the most rigorous fairness treatment in timetabling.
  Directly quotable finding: **"fairness can often be improved at the cost
  of only a small increase in the overall amount of penalty."**
- **Burget & Rudová**, "Teacher-oriented Fairness in Course Timetabling,"
  PATAT 2016 **[REF-OK]** — analogous to our stage B.
- **"The (a)social exam invigilator assignment problem"**, *Journal of
  Scheduling* (2025) **[REF-OK]**. Notes the standard practical difficulty:
  a perfectly fair split of duties is usually fractional, so someone does 4
  while others do 3 — a lexicographic tie-break, which is what §6 is.

---

## 4. The rigour upgrade: our objective is already *equitable*

**Anchor to Ogryczak's equitable optimisation.** Ogryczak, W., Luss, H.,
Pióro, M., Nace, D., & Tomaszewski, A. (2014). "Fair Optimization and
Networks: A Survey." *Journal of Applied Mathematics*, 2014:612018.
**[REF-OK / V-SNIP]**

An objective is **equitable** if it is symmetric and satisfies the
**Pigou–Dalton principle of transfers** — moving burden from a less-loaded
party to a more-loaded one never improves it. Equivalently: **Schur-convex**.

**Our objective already qualifies.** A symmetric sum of a convex increasing
function is Schur-convex, so `Σ_s φ(excess_s)` with `φ = max(0,·)^p`, p ≥ 1
satisfies Pigou–Dalton on the excess vector.

> This converts *"we squared it because it felt right"* into *"the
> aggregation is Schur-convex and satisfies the Pigou–Dalton transfer
> principle, placing it in the equitable-optimisation family of Ogryczak et
> al. (2014)."* Same code, materially better defence.

Related, if pressed on why this fairness measure and not another: Lan, Kao,
Chiang & Sabharwal, "An Axiomatic Theory of Fairness in Network Resource
Allocation," INFOCOM 2010 **[REF-OK]** — five axioms generating a family
containing α-fairness, Jain's index and entropy as special cases.

### The exponent, positioned

α-fairness (Mo & Walrand 2000) interpolates: **α=0 utilitarian, α=1
proportional, α→∞ max-min**. **[V-SNIP]** Our `p` plays the same role.

| p | Behaviour | Attack |
|---|---|---|
| 1 | Utilitarian (ℓ₁) | Indifferent to concentration — will cram one student to marginally relieve many. Lay-fatal. |
| **2** | **Quadratic (ℓ₂), the standard compromise** | — |
| →∞ | Minimax (ℓ_∞) | **Hostage-taking**: one structurally impossible student fixes the value and the model goes blind to everyone else. This is the standard motivation for leximin over plain maximin. |

ℓ_p load balancing is described as *"one of the basic and fundamental
objectives investigated in scheduling theory"* **[V-SNIP]**, and personnel
scheduling justifies quadratic penalties perceptually: *"small deviations
are generally acceptable, larger imbalances cause sharp increases in
dissatisfaction"* **[V-SNIP]**, from *Applied Sciences* 16(4):1747
**[REF-OK]** — check before quoting.

**Honest gap: no study measures how `p` affects human fairness
perception.** If asked "why 2 and not 3?", the answer is a sensitivity
analysis, not evidence.

**Our penalty has a textbook name:** penalising only positive deviation from
a per-agent target is **one-sided goal programming**; squared, it is
**quadratic goal programming**. **[RECALL — terminology mapping is standard
textbook material but was not verified in-session.]**

---

## 5. Lexicographic staging is standard and named

Our two-stage design is **lexicographic (preemptive) optimisation**, also
called hierarchical optimisation, or in goal programming **preemptive /
non-Archimedean**. Solving level 1 then adding `Z = Z*` as a constraint is
the standard **bounded-objective** realisation. **[V-SNIP]**

It is **easier to defend to a lay audience than a weighted sum**, because it
makes an unambiguous promise: *no teacher convenience was ever traded
against a learner's timetable.* A weighted sum cannot promise that — and
once the weights are published, someone will compute the exchange rate and
be offended by it.

**Two pitfalls that will bite:**

1. **The tie set may be empty of anything useful.** The universal criticism
   of preemptive goal programming is that lower levels frequently have no
   influence — the higher level pins the solution and the rest is
   decoration. **Must be measured** (`DESIGN.md` §4.0.3).
2. **Pinning `Z = Z*` exactly is brittle.** A timetable worse for students
   by one unit of squared excess — invisible to any human — but far better
   for teachers is excluded by construction. The standard fix is
   **ε-relaxation**, and search results describe it as established:
   relaxing the constraint *"allows additional sections of the Pareto front
   to be explored that may have been unobtainable using the stricter
   lexicographic method."* **[V-SNIP]**

**There is no established name for "deliberately leave ties so stage 2 has
freedom."** Searched specifically; do not invent one. Describe the design as
**"lexicographic optimisation with a secondary objective resolved over the
set of alternative optima at the primary level, with an optional
ε-tolerance."** Related recognised concepts: ε-constraint method,
alternative optima / solution multiplicity, solution pools, secondary
tie-breaking objective, and **Modelling to Generate Alternatives (MGA)** —
which is arguably the better framing (§7).

---

## 6. Stress factors and the evidence on spacing

**This is the weakest-evidenced part of the design. Candour here buys
credibility elsewhere.**

**No exam-timetabling paper weights exams by difficulty or stress.**
Searched four ways. The literature weights by **number of students**
(a size proxy), **duration** (for room-mixing only, never as student
burden), and **size for front-loading**. `SF` is therefore a genuine
extension, not a standard technique.

**Frame it as ITC-2007 does its own weights: declared institutional
policy** (§1). And **never call `SF` "difficulty"** — call it a *declared
relative preparation-and-recovery burden*, set by the people qualified to
judge, governed and versioned like policy rather than presented as
measurement.

### The one study that matches our setting

**Goulas, S., & Megalokonomou, R. (2020). "Marathon, Hurdling, or Sprint?
The Effects of Exam Scheduling on Academic Performance." *B.E. Journal of
Economic Analysis & Policy*, 20(2).** DOI 10.1515/bejeap-2019-0177.
**[V-SNIP / REF-OK]**

A Greek public **high school** — our institutional setting, not a
university — with schedules generated by **lottery**, giving near-
experimental identification. It separates three channels: **days between
exams** (rest), **days since the first exam** (cumulative fatigue,
negative), and **exam order** (warm-up, positive). Effects stronger in STEM.

> **This is direct empirical support for a window-based rather than a purely
> pairwise formulation** — the two channels it identifies are exactly what
> sliding-window load measures and what Carter's pairwise proximity cost
> cannot see. Worth a paragraph in any write-up.

**The number not to hide:** optimising the schedule improves performance by
**~0.02 standard deviations**. **[V-SNIP]** Small. **Do not build the case
on results.** Build it on burden, equity and defensibility.

Supporting: *Education Sciences* 12(2):94 (2022) **[V-SNIP for content,
REF-OK for citation — author list unverified]**, a natural experiment where
exams moved from 7–14 days apart to back-to-back with a 15-minute break;
performance and satisfaction both declined. Caveats to state before a critic
does: single institution, medical students, confounded with curricular
restructuring. Also Di Pietro (2013), *Bulletin of Economic Research*
**[REF-OK]**.

Cepeda et al. (2006) on distributed practice **[V-SNIP]** is about *study*
spacing, not exam spacing. Suggestive background only — **do not overclaim
it**, someone will call it out.

### What actually governs practice: hard caps

Real institutions do not write convex penalties into policy. They write
**hard, human-readable thresholds** — all **[V-SNIP]**:

- **NC State:** no three consecutively scheduled exams within 24 hours.
- **Cornell:** minimise students with **more than two exams in one 24-hour
  period**, concretely defined.
- **Wake Forest:** no more than two in 24 hours; beyond that, reschedulable.
- **Florida:** no more than three in one day. **Duke:** petition right.
- **UniTime:** a first-class criterion named "more than two exams a day".

> **The convex objective is the right thing to optimise and the wrong thing
> to promise.** Governing bodies understand *"no learner writes three papers
> in two days."* They do not understand *"we minimised the sum of squared
> normalised window excesses."* We need both — see `DESIGN.md` §2.5.

### South African context

No published DBE technical criteria for timetable construction were
obtainable. Secondary sources describe the NSC timetable as *"balancing
cognitive load while allowing time between major papers"* over seven weeks
with built-in non-examination days. **[V-SNIP — journalistic/secondary
only.]**

Two consequences: the national examining authority's **own stated intent is
cognitive-load balancing**, which is more persuasive to a school governing
body than any OR citation; and the DBE publishes no formal model, so there
is **no national standard we can be accused of deviating from**.

**Highest-value single addition to this review:** the official DBE
examination-administration policy (April 2019 regulations), which could not
be accessed. Also relevant: the DBE Accommodations and Concessions
procedural manual **[REF-OK]** — concession candidates are a stakeholder
group the design currently does not model.

---

## 7. Terminology to adopt

Where the design stops reading as ad hoc.

| Say this | Instead of |
|---|---|
| **"Institutional weightings"** (ITC-2007's own term) | "our stress factors" |
| **"Declared relative preparation-and-recovery burden"** | "difficulty" |
| **"Per-student proportional entitlement / fair-share baseline"** | "their own baseline" |
| **"One-sided positive deviation from a goal"** (goal programming) | "max(0, excess)" |
| **"Quadratic (ℓ₂) penalty on normalised excess"** | "we square it" |
| **"Schur-convex; satisfies Pigou–Dalton; equitable per Ogryczak et al. (2014)"** | "fairer than a plain sum" |
| **"Lexicographic optimisation with a secondary objective over the alternative optima, with optional ε-tolerance"** | "two-stage, we freeze stage A" |
| **"Certificate of optimality / zero optimality gap"** | "proven optimal" |
| **"Provably optimal with respect to the published fairness criterion"** | **"the optimal timetable"** |
| **"Price of fairness"** (Bertsimas, Farias & Trichakis 2011) | "what fairness cost us" |
| **"CVaR at 10% — average excess among the worst-off tenth"** | "the bad cases" |

Two sentences worth having ready.

**For an OR reader:** *"The student objective is a Schur-convex, one-sided
quadratic goal-programming aggregation of per-student normalised window
excess, minimised to proven optimality; teacher concerns are resolved
lexicographically over the set of alternative optima, with a published
ε-tolerance."*

**For a parent or governing body:** *"We wrote down, in advance and in
public, what a fair spread of papers means for every individual learner
relative to their own subject load; we then had the computer search every
possible timetable and prove that none is fairer by that standard; and we
publish who came off worst, so you can judge the standard for yourself."*

---

## 8. Where we stand relative to the literature

| Our choice | Literature status |
|---|---|
| Window-load measure over sliding 2–5 day windows | **Non-standard** (the field is pairwise) but **empirically better supported** — Goulas & Megalokonomou separate rest from cumulative fatigue, which pairwise cannot express |
| Per-student normalisation `Λ_s·w/D_s` | **Found nowhere, criticised nowhere** — searched four ways. An unusual but favourable position: filling a gap the fairness literature has explicitly named |
| Convex one-sided penalty, p=2 | **Standard** in load balancing and personnel scheduling; Schur-convex, hence equitable |
| Lexicographic student → teacher | **Standard**; Muklason et al. use a three-stage variant in this exact domain |
| Proven optimality via CP-SAT | **Current best practice** — Cornell and Bucknell are both IP-based |
| `SF` as a stress weight | **A genuine extension.** Frame as declared institutional policy, exactly as ITC-2007 frames its own weights |
| Hard human-readable caps | **Universal in real institutional policy**, and previously missing from our design |
