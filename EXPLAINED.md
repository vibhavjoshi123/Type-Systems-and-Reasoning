# The overtime exemption, explained

Everything in one place: the problem, the data model, the rulebook, what the
database settles, what the model settles, a worked determination traced call
by call, and the measurements.

Every figure here came from a run against a live TypeDB instance and live
calls to a decision model.

1. [The problem](#1-the-problem)
2. [The data model](#2-the-data-model)
3. [The 19 tests a function settles](#3-the-19-tests-a-function-settles)
4. [The rulebook as a tree](#4-the-rulebook-as-a-tree)
5. [What goes to the model, and why](#5-what-goes-to-the-model-and-why)
6. [The search](#6-the-search)
7. [One determination, traced](#7-one-determination-traced)
8. [Measurements](#8-measurements)
9. [How a reviewer gets evidence](#9-how-a-reviewer-gets-evidence)

---

## 1. The problem

In the US most employees must be paid overtime beyond 40 hours a week. Some
are **exempt**: no overtime however many hours they work.

Getting it wrong is expensive in both directions. Classify someone exempt
who is not and you owe years of back pay. The question is litigated
constantly, and almost always over the same thing — not the salary, which
nobody disputes, but what the person actually *does*.

**29 USC § 213(a)** creates the exemption. **29 CFR Part 541** defines it.
Five categories, any one sufficient:

| category | regulation | turns on |
|---|---|---|
| Executive | § 541.100 | managing a department; hiring and firing |
| Administrative | § 541.200 | business operations; discretion on significant matters |
| Professional | § 541.300 | advanced knowledge, or creative originality |
| Computer | § 213(a)(17) | systems analysis, design, programming |
| Outside Sales | § 541.500 | making sales away from the premises |

Plus two gates that apply regardless: the employer must clear the
**§ 203(s)** revenue floor of $500,000, and some roles are **never** exempt
however they are paid — **§ 541.3** names blue-collar work, police, fire
fighters and paramedics.

---

## 2. The data model

Seven entities, six relations, 30 attributes. Three choices in it are
load-bearing.

The employee records used throughout are a synthetic fixture of 25 people,
each built to exercise a particular clause — a teacher under the salary
threshold for § 541.303, a help-desk technician for § 541.401's exclusion, a
paramedic for § 541.3's categorical bar.

### A missing description is legal

```typeql
entity employee,
    owns emp_id @key,
    owns job_title @card(1),
    owns job_description @card(0..1),     # 0..1, deliberately
    owns weekly_salary @card(0..1),
    owns hire_fire_authority @card(0..1),
    ...
```

`@card(0..1)` means an employee may have no description on file. That is not
an oversight. § 541.2 says a title establishes nothing, so with no
description the duty tests cannot be settled *either way* — and the system
must be able to say so rather than guess. Section 7 shows what that produces.

### Classifications live in data, not in code

```typeql
entity occupation,
    owns occupation_code @key,
    owns name @card(1),
    owns classification @card(0..);       # multi-valued
```

So § 541.3's categorical bar, § 541.401's exclusion of help desk and
hardware repair, and § 541.303's salary-test exemption for teachers and
lawyers are all **rows**. When the DOL moves an occupation between
categories that is an `insert`, not a schema migration and not a code change.

### Supervision is one n-ary relation, counted in FTE

```typeql
relation supervision,
    relates supervisor @card(1),
    relates supervised @card(1..);        # the whole span, not one edge each
```

§ 541.104 counts **full-time equivalents**, not heads — two half-time staff
is 1.0 FTE, not two people. Getting that wrong is a real misclassification,
so the sum is over the relation's role players and the rule is written once.

---

## 3. The 19 tests a function settles

Five shapes cover all of them.

**Threshold over a stored number.** Reused by four separate goals, because
four places in the regulation cite the same figure:

```typeql
fun meets_salary_threshold($e: employee) -> boolean:
  match
    $e has weekly_salary $s;
    $s >= 684.0;                          # § 541.600(a)
  return check;
```

**Reading a boolean somebody recorded:**

```typeql
fun paid_on_salary_basis($e: employee) -> boolean:
  match $e has salary_guaranteed true;    # § 541.602
  return check;
```

**A classification lookup** — the pattern five goals use:

```typeql
fun is_categorically_nonexempt($e: employee) -> boolean:
  match
    holds_occupation (worker: $e, occupation: $o);
    $o has classification "categorically_nonexempt";   # § 541.3
  return check;
```

**Aggregate, then threshold** — split in two on purpose, so the § 541.104
FTE rule is stated once and the threshold separately:

```typeql
fun supervised_fte($e: employee) -> double:
  match
    supervision (supervisor: $e, supervised: $p);
    $p has fte $f;
  return sum($f);

fun directs_two_or_more_fte($e: employee) -> boolean:
  match
    let $total in supervised_fte($e);
    $total >= 2.0;
  return check;
```

**Set membership**, written as a disjunction of equalities because the list
form is rejected for an attribute variable:

```typeql
fun holds_qualifying_credential($e: employee) -> boolean:
  match
    credential_held (holder: $e, credential: $c);
    $c has credential_level $l;
    { $l == "bachelor"; } or { $l == "master"; } or
      { $l == "doctorate"; } or { $l == "professional_license"; };
  return check;                           # § 541.301(d)-(e)
```

### Why this matters beyond correctness

Functions are evaluated on demand. There is no stored result, so there is
nothing to invalidate. Change one rating or one salary and the next call
reflects it, with no recompute step and no cache.

And the thresholds live in the schema rather than in a program. Changing
$684 is one `redefine` and every caller follows — the nightly job, an ad-hoc
query, a dashboard. Compute it in your own code instead and the rule exists
in two places, and one day they disagree.

---

## 4. The rulebook as a tree

```
exempt_from_overtime                      AND
├── enterprise_covered                    db   0.3    § 203(s)(1)(A)(ii)
├── not_categorically_nonexempt           db   0.4    § 541.3
├── an_exemption_applies                  OR   2.0
│   ├── outside_sales                     AND  7.1    § 541.500
│   │   ├── os_regularly_offsite          db   1.1    § 541.502
│   │   └── os_primary_duty_is_sales      MODEL 6.0   § 541.500(a)(1)
│   ├── computer_employee                 AND  9.0    § 213(a)(17)
│   │   ├── ce_occupation_qualifies       db   0.4    § 541.400(b)
│   │   ├── ce_not_excluded_role          db   0.5    § 541.401
│   │   ├── ce_compensation               OR   1.0
│   │   │   ├── ce_hourly_threshold       db   0.8    $27.63/hr
│   │   │   └── ce_salary_threshold       db   0.9    $684/wk
│   │   └── ce_primary_duty_qualifies     MODEL 6.5   § 213(a)(17)(A)-(D)
│   ├── executive                         AND  10.3   § 541.100
│   │   ├── ex_salary_basis               db   0.7    § 541.602
│   │   ├── ex_salary_threshold           db   0.8    § 541.600(a)
│   │   ├── ex_directs_two_or_more        db   1.2    § 541.104
│   │   ├── ex_hiring_influence           OR   2.0    § 541.100(a)(4)
│   │   │   ├── ex_hire_fire_authority    db   0.6
│   │   │   └── ex_recommendations_weighted MODEL 6.5 § 541.105
│   │   └── ex_primary_duty_is_management MODEL 7.0   § 541.100(a)(2)
│   ├── administrative                    AND  17.5   § 541.200
│   │   ├── ad_salary_basis               db   0.7
│   │   ├── ad_salary_threshold           db   0.8
│   │   ├── ad_primarily_non_manual       db   1.0    § 541.200(a)(2)
│   │   ├── ad_directly_related_to_mgmt   MODEL 7.0   § 541.201
│   │   └── ad_discretion_and_judgement   MODEL 8.0   § 541.202
│   └── professional                      AND  18.0   § 541.300
│       ├── pr_salary_requirement         OR   1.2
│       │   ├── pr_exempt_from_salary_test db  0.5    § 541.303-304
│       │   └── pr_salary_met             AND  1.6
│       │       ├── pr_salary_basis       db   0.7
│       │       └── pr_salary_threshold   db   0.8
│       └── pr_professional_duty          OR   2.0
│           ├── pr_creative               AND  7.0
│           │   ├── pr_creative_field     db   0.6    § 541.302(a)
│           │   └── pr_invention_...      MODEL 7.0   § 541.302
│           └── pr_learned                AND  8.5
│               ├── pr_credential         db   0.9    § 541.301(d)-(e)
│               ├── pr_advanced_knowledge MODEL 7.5   § 541.301(a)-(c)
│               └── pr_consistent_discretion MODEL 8.0 § 541.301(b)
└── duties_match_job_description          MODEL 7.0   § 541.2
```

**29 leaves. 19 database. 10 model. Depth 5.**

Three structural features worth naming, because each one corresponds to
something real in the regulation:

**A nested OR where both sides are free.** A computer employee qualifies on
*either* $27.63/hour *or* $684/week — § 213(a)(17) allows both, and both are
stored numbers.

**A nested OR where one side is 10× the other.** `ex_hiring_influence`:
either you have hire/fire authority (a recorded boolean, cost 0.6) *or* your
recommendations are "given particular weight" (§ 541.105, a judgement, cost
6.5). The search should prefer the cheap side when it can.

**Depth 5 under `professional`**, because the regulation really is that
shaped. A teacher is exempt from the salary tests *entirely* under § 541.303
— so the salary requirement is a disjunction, not a plain AND. Build it as
an AND and a teacher earning $620 comes out wrong.

Note also that the cost column is monotone in model-call count:
`outside_sales` at 7.1 has one judgement, `professional` at 18.0 has three.
That turns out to matter in section 8.

---

## 5. What goes to the model, and why

All ten judgement goals send exactly two things: `job_title` and
`job_description`. Nothing else.

```
os_primary_duty_is_sales                 § 541.500(a)(1)
ce_primary_duty_qualifies                § 213(a)(17)(A)-(D)
ex_recommendations_weighted              § 541.105
ex_primary_duty_is_management            § 541.100(a)(2)
ad_directly_related_to_management        § 541.201
ad_discretion_and_independent_judgement  § 541.202
pr_invention_imagination_originality     § 541.302
pr_advanced_knowledge                    § 541.301(a)-(c)
pr_consistent_discretion                 § 541.301(b)
duties_match_job_description             § 541.2
```

### The dividing line, exactly

| the database reads | the model reads |
|---|---|
| a **number** — salary, FTE, duty share, revenue | **prose** |
| a **boolean** somebody recorded | — |
| a **classification** somebody assigned | — |
| a **count or sum** over relations | — |

Every one of the 19 resolves to comparing a stored value. Not one reads a
sentence. And every one of the 10 is the same question in different forms:
**what is this person actually doing?**

### The two hardest, which are the point

§ 541.201 and § 541.202 both sit in the administrative branch, and they are
different questions about the same text:

- **§ 541.201** — is the work *about running the business*, as distinct from
  *producing what the business sells*? (The "administrative/production
  dichotomy", the most litigated line in the whole regulation.)
- **§ 541.202** — does the person *decide things of significance*, or follow
  a procedure?

A clerk following a documented filing procedure arguably passes the first
and clearly fails the second. No formula separates them. So the model is
asked one atomic question at a time rather than "is this person
administrative".

### How the answer comes back

One number: the probability of yes.

```
0.95  →  almost certainly yes
0.50  →  no information
0.03  →  almost certainly no
```

Confidence is derived from the position rather than self-reported:

```
confidence = 2 × |p − 0.5|
```

That formula is not an invention — it is the vendor's own confidence
statistic for a choice, specialised to two options. Their documentation
confirms the premise too: a yes/no answer carries no confidence field,
because for a binary question the probability *is* the uncertainty.

It matters because self-reported confidence carries no signal. Measured on
a general chat model across the same cases: mean **0.92 when right and 1.00
when wrong** — a negative separation. A 0.50 here cannot pretend to be sure.

### Three outcomes, not two

```
confident yes  (p > 0.5, confidence ≥ 0.6)  →  the test is satisfied
confident no   (p < 0.5, confidence ≥ 0.6)  →  not satisfied; the search backtracks
anything else                                →  recorded as a GAP
```

The third row is the reason a model is here rather than a yes/no classifier.
A binary gate must pick one, and both choices are wrong: call it *no* and
you deny an exemption because a document was not uploaded; call it *yes* and
you grant one nobody checked.

---

## 6. The search

Three tactics, tried in this order. **The order is the heuristic.**

| tactic | effect |
|---|---|
| `split` | a compound test becomes its parts |
| `schema-function` | ask the database — exact, free |
| `ask-model` | ask the model — costs money, returns a probability |

```
prove(goal)
  for each tactic in order:
    if it does not apply: skip
    snapshot the state
    run it
      ├─ produced subtests?  all must hold (AND) / any suffices (OR)
      │     success → settle, discharge the siblings that became moot
      │     failure → restore the snapshot, try the next tactic
      ├─ satisfied          → settle
      ├─ cannot tell        → settle AND record a gap
      └─ not satisfied      → restore the snapshot, try the next tactic
```

**Within an AND, cheapest first, and the first failure decides.** That is
why the expensive judgements usually never run: a branch that fails any
cheap database test is abandoned before reaching them.

**Within an OR, the state is restored between attempts** — that restore *is*
the backtracking.

**And an OR prefers a clean close over a gappy one.** This was a defect
found by running five different orderings and comparing them. Accepting the
first branch that closed — including one that closed only by admitting a gap
— meant an employee qualifying cleanly under Executive and
only-modulo-a-gap under Administrative came back **exempt** or **needs
review** according to which was tried first. Ordering was changing the
answer. A gappy success is now held in reserve while the search looks for a
clean route, and taken only if none exists.

---

## 7. One determination, traced

The subject:

```
job title       Senior Staff          ← establishes nothing (§ 541.2)
weekly salary   $2,600
supervises      3.0 full-time equivalents
can hire/fire   true
description     "Runs the regional distribution department. Sets its budget
                 and staffing plan, interviews and hires its twelve
                 employees, conducts their performance reviews and decides
                 promotions and dismissals. Spends the working day directing
                 the department rather than performing its work."
```

### The run

```
[01] split exempt_from_overtime [ALL must hold]
[02] db    enterprise_is_covered($e)        = True
[03] db    is_categorically_nonexempt($e)   = False  -> negated, goal is True
[04] db    fetch job_title + job_description        (one read, cached)
[05] MODEL which branch first?  → executive
[07] split an_exemption_applies [ANY suffices]
[08] split executive [ALL must hold]
[09] db    paid_on_salary_basis($e)         = True
[10] db    meets_salary_threshold($e)       = True    $2,600 ≥ $684
[11] db    directs_two_or_more_fte($e)      = True    3.0 ≥ 2.0
[12] MODEL which hiring route first? → ex_hire_fire_authority
[15] db    has_hire_fire_authority($e)      = True
[16] MODEL "is primary duty management?"    § 541.100(a)(2)
              p=0.95  confidence=0.90  → satisfied
[17] MODEL "do the duties match the title?" § 541.2
              p=0.76  confidence=0.46  → below the bar, recorded as a GAP
```

### The result

```
verdict            needs human review
safe to act on     False
database queries   6 formula checks + 1 evidence read
model requests     2 proposals + 2 judgements
cost               $0.000091
```

Four things in that trace are the whole design working:

**Six of eight decisions never touched a model.** The salary, the FTE count,
the hire/fire flag, the revenue floor and the categorical bar are all exact.

**Four of five exemptions were never examined.** The branch proposal put
Executive first and it closed, so Administrative, Outside Sales, Computer
and Professional — carrying 7 judgements between them — were never entered.

**At the nested hiring choice the model chose the database route.** § 541.105
"particular weight" is a judgement; hire/fire authority is a stored boolean.
It picked the boolean, costing a `True` instead of a model call.

**The last judgement produced a gap, and the gap changed the verdict.**
`0.76` with confidence `0.46` is a lean, not a finding — reasonably, since
the title is "Senior Staff" and there is little to match duties against. So
the determination comes back *needs human review* rather than *exempt*, and
says exactly which clause a human must settle.

---

## 8. Measurements

### Remove the model, and nothing can be cleared

25 employees, run twice — once with the decision model, once with it gone.

```
                 exempt   needs review   not exempt
with the model      14          6              5
without it           0         22              3
```

**Zero exempt determinations without it.** Every exemption branch ends in a
judgement, so *exempt* is unreachable. The database alone gets you to
"everything measurable checks out, and I cannot tell whether they qualify".

The three that survive are all **negative** — § 541.3 barring a line
operative and a paramedic, § 203(s) failing a small employer. A type system
can rule people **out** by itself. It cannot rule anyone **in**.

Two also go the *wrong* way without it: a filing clerk and a help-desk
technician are correctly **not exempt** with the model (failing § 541.202
and § 541.401 respectively) and become "ask a human" without it.

### Cost at scale

```
1,000 employees → ~2,667 judgements → $0.042
```

Four cents, for determinations that have no formula and that a person would
otherwise make one at a time.

### Should the model also choose the branch?

It was built and measured. Fifteen employees constructed so that three
branches stay live for every one — all salaried at $2,600, non-manual,
supervising 3 FTE, credentialed, with identical titles. Only the description
differs. Metric: judgement tests reached.

```
ordering                        judgements   vs cost   proposals
perfect (ceiling)                    40        −15         0
model, shown the description         40        −15        30
model, branch labels only            51         −4        26
random                               50         −5         0
cost annotation (the default)         55          —         0
worst possible                        60         +5         0
```

**It chooses perfectly — 15 of 15 — and still should not do it.**

A proposal sends 735 input tokens; a judgement sends 375. So one proposal
must avoid **1.96** judgements to break even. Measured, it avoided **0.5**.
Four times short, at any scale.

The reason is the architecture working as intended: the cheap database tests
run first inside every branch, so **a wrong branch is already cheap to
discover**. The thing a proposer exists to avoid has been made inexpensive by
the type system.

Two further results from that table, both of which correct earlier
assumptions:

**Shown only branch labels, the model scored like random** (51 vs 50). A
proposer denied the evidence its branches turn on is guessing. It needs the
job description — which is now derived automatically from the branches' own
declarations of what they require, rather than configured.

**The cost annotation is fragile.** On an earlier fixture it was nearly
optimal (58 against a perfect 55). Here it is **second worst**, beaten by
random. Sorting by cost only works while cost predicts which branch wins;
once it does not, it is worse than chance.

---

## 9. How a reviewer gets evidence

A determination is only auditable if someone who was not there can
reconstruct it from the **database**, with the process long exited. So it is
stored as a relation that links what it relied on as role players:

```typeql
relation determination,
    relates about @card(1),           # the employee
    relates relied_on @card(0..),     # every test that was settled
    relates admitted @card(0..),      # the ones nobody could settle
    owns verdict @card(1),
    owns decided_at @card(1),
    owns full_trace @card(1);
```

Each test becomes a finding carrying its name, whether it held, **how** it
was settled, its authority, and — if a model answered — the probability and
confidence.

Four questions, each answered by a query:

**What was decided?**
```
Contested executive 0  (Senior Staff)
verdict: needs_human_review     decided 2026-10-12 by typedb+decision-model
```

**On what basis?**
```
test                             held        settled by   authority
enterprise_covered               True   schema-function   § 203(s)
not_categorically_nonexempt      True   schema-function   § 541.3
ex_salary_basis                  True   schema-function   § 541.602
ex_salary_threshold              True   schema-function   § 541.600(a)
ex_directs_two_or_more           True   schema-function   § 541.104
ex_hire_fire_authority           True   schema-function   § 541.100(a)(4)
ex_primary_duty_is_management    True         ask-model    § 541.100(a)(2)
```

The middle column is the point: six were computed from stored data, one was
a model reading prose. Those warrant different scrutiny and the record says
which is which.

**What could nobody settle?**
```
duties_match_job_description   [§ 541.2]
   answered 0.76 at confidence 0.52, below the bar
   ADMITTED, not decided — a human must settle it
```

Stored in a **separate role**, so "find every determination resting on an
assumption" is one query rather than a log search.

**How sure was the model?**
```
duties_match_job_description     p=0.76   confidence=0.52
ex_primary_duty_is_management    p=0.95   confidence=0.90
```

### Links, not copies

The record does not store "salary was $2,600". It links the findings, and
the queries read the salary off the employee. A copied number would quietly
disagree with reality after a correction; a link can be re-read and checked.

### Three gaps, stated

**A finding does not name the row it read.** `ex_salary_threshold: True` is
recorded without "because weekly_salary was 2600 when I looked". Edit the
salary later and nothing flags the drift.

**Re-running does not link to the prior determination.** It should attach via
a supersession morphism rather than sitting alongside. The machinery exists;
it is not wired in here.

**The reviewer's own answer is not captured.** A human settles § 541.2 and
that should become a finding marked `settled_how: "human"`, closing the gap.
Nothing writes it back yet.
