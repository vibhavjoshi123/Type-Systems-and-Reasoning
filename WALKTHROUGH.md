# Walkthrough

The technical account. For the version that assumes no background, see
[EXPLAINED.md](EXPLAINED.md).

Every figure here came from a run against a live TypeDB instance and live
calls to a decision model.

1. [The problem](#1-the-problem)
2. [The data](#2-the-data)
3. [The build, in dependency order](#3-the-build-in-dependency-order)
4. [How the functions depend on each other](#4-how-the-functions-depend-on-each-other)
5. [The policy as a goal tree](#5-the-policy-as-a-goal-tree)
6. [The search](#6-the-search)
7. [The model protocol](#7-the-model-protocol)
8. [Compared against language models](#8-compared-against-language-models)
9. [The judge](#9-the-judge)
10. [What this shows and what it does not](#10-what-this-shows-and-what-it-does-not)

---

## 1. The problem

A fund's written policy, as three requirements, all of which must hold:

```
IPS 2.1   covered by at least two distinct analysts
IPS 2.2   clears the quality bar, by EITHER route:
            (a) mean rating ≥ 4.0  AND  analysts not materially split
            (b) one analyst at 5   AND  at least three analysts covering
IPS 3.1   the issuer's disclosed business is consistent with its sector
```

Applying it nightly to a thousand holdings requires answering three
questions the next morning: what was decided and on what basis, what was not
known, and whether the chain can be trusted. A system where the component
that reasons is also the component that decides what gets stored cannot
answer the third.

---

## 2. The data

Securities, companies, exchanges, indices, currencies, banks, analysts and
price bars. Eight records drive everything, each one saying *this analyst
rates this security N out of 5*:

| | Jane Doe | John Smith | Maria Garcia | mean | spread |
|---|---|---|---|---|---|
| AAPL | 5 | 4 | 4 | 4.33 | 1 |
| NVDA | 5 | 5 | — | 5.00 | 0 |
| TSLA | 3 | 2 | 4 | 3.00 | 2 |

The rating is not an attribute of the analyst or of the security. It is an
attribute of **the act of covering**:

```typeql
relation coverage,
  relates covered @card(1),
  relates analyst @card(1),
  owns rating;
```

That shape is load-bearing. A verdict later needs to point at the specific
ratings behind it, and you can point at a relation. A number flattened into
a column alongside twenty others cannot be pointed at.

Descriptions are declared `@card(0..1)` — optional:

```
AAPL   "Designs and sells consumer electronics, software and services."
NVDA   "Designs GPUs and accelerated-computing platforms."
TSLA   (nothing on file)
```

TSLA's gap is the most informative case here; §7 returns to it.

---

## 3. The build, in dependency order

Four layers, each usable before the next exists.

**Layer 1 — the base model, unmodified.** Entity and relation types with
their cardinality constraints, plus four functions: `latest_price`,
`market_cap`, `constituent_count`, `consensus_rating`.

**Layer 2 — provenance types.** 11 attributes and 4 relation types, applied
as pure `define`. The centre of it:

```typeql
relation derived_claim,
    relates about @card(1),            # the security the claim concerns
    relates from_price @card(0..),     # the price bars that fed it
    relates from_coverage @card(0..),  # the coverage relations that fed it
    owns claim_id @key,
    owns decision,                     # @values("include","review","reject")
    owns prong_trace,
    ...;

coverage plays derived_claim:from_coverage @card(0..);
claim_morphism plays claim_morphism:earlier @card(0..);
```

Two lines do the work. The first lets a claim point at the individual
coverage relations it averaged — a relation playing a role in another
relation. The second makes the correction hierarchy self-referential, so a
restatement of a restatement is expressible to any depth.

The inputs are **role players, not copied values**, so a claim cannot drift
from its evidence.

**Layer 3 — derived functions.** Fourteen, in a separate `define` block from
the types so a bad function body cannot abort the type load. §4 is their
dependency graph.

**Layer 4 — the policy, as a goal tree.** Ordinary data, declared separately
from the procedure that walks it. §5.

Nothing above Layer 1 modifies anything below it. The base model's four
functions return the same answers after all of it.

---

## 4. How the functions depend on each other

Eighteen functions. Eleven read data directly; seven are built on others.
This table is derived from the function bodies, not drawn by hand.

| function | origin | returns | depends on |
|---|---|---|---|
| `latest_price` | base | `double` | — reads price bars |
| `consensus_rating` | base | `double` | — reads coverage |
| `constituent_count` | base | `integer` | — reads index membership |
| `market_cap` | base | `double` | `latest_price` |
| `coverage_breadth` | added | `integer` | — reads coverage |
| `rating_spread` | added | `integer` | — reads coverage |
| `has_strong_conviction` | added | `boolean` | — reads coverage |
| `venue_count` | added | `integer` | — reads listings |
| `uncovered_constituents` | added | `{ equity }` | — reads index membership |
| `meets_coverage_floor` | added | `boolean` | `coverage_breadth` |
| `has_broad_coverage` | added | `boolean` | `coverage_breadth` |
| `meets_consensus_floor` | added | `boolean` | `consensus_rating` |
| `is_contested` | added | `boolean` | `rating_spread` |
| `index_market_cap` | added | `double` | `market_cap` |
| `index_weight` | added | `double` | `market_cap`, `index_market_cap` |
| `current_claims` | added | `{ derived_claim }` | — reads claims |
| `current_decision` | added | `{ derived_claim }` | — reads claims |
| `claim_inputs` | added | `{ price_bar }` | — reads claim role players |

### The two patterns

**Threshold over a measure.** A measure function returns a number; a
predicate function wraps it with the policy's threshold and returns a
boolean. This is how the policy gets into the schema:

```typeql
fun coverage_breadth($s: security) -> integer:
  match
    $cov isa coverage, links (covered: $s, analyst: $a);
  select $a;
  distinct;                                   # each analyst counted once
  return count;

fun meets_coverage_floor($s: security) -> boolean:
  match
    let $n in coverage_breadth($s);           # calls the measure
    $n >= 2;                                  # IPS 2.1's threshold
  return check;                               # true if satisfiable
```

Two predicates reuse one measure — `meets_coverage_floor` (≥ 2, IPS 2.1) and
`has_broad_coverage` (≥ 3, IPS 2.2(b)(ii)) both call `coverage_breadth`. One
definition of "how many analysts", two thresholds over it.

**Composition over an aggregate.** `index_weight` is four levels deep and
the only place in the example where depth matters:

```
index_weight($s, $i)                    AAPL's share of the Nasdaq 100
  ├── market_cap($s)                       AAPL's value
  │     └── latest_price($s)                  most recent price bar
  └── index_market_cap($i)                 the index's total value
        └── market_cap($s) for each         …per constituent
              └── latest_price($s)
```

### Why this matters: nothing is stored

TypeDB functions are evaluated on demand. There is no materialised column
and no cache, so there is nothing to invalidate. Observed:

```
one coverage relation rewritten, no recompute step run:
  consensus_rating(AAPL)      4.3333  →  3.3333
  meets_consensus_floor(AAPL)   True  →  False

one new NVDA price bar inserted:
  index_weight(AAPL)          77.82%  →  53.38%
```

AAPL's own figures did not change in the second case. The index grew, so
AAPL is a smaller share of it — and the right answer surfaced four levels up
with nothing told to update. With stored values you would need a dependency
system that knows a price bar for any constituent touches every other
constituent's weight. That system is where bugs live; here it does not need
to exist.

### And the cost of it

```
one boolean function call    251.7 ms
one raw ratings fetch        254.8 ms
```

A function call costs about what a plain query costs, so **the price is
round trips** — and on that count functions lose. The search makes three to
five per security where a program could fetch the ratings once and compute
all five predicates locally.

What that buys is one definition of the policy instead of two. Changing the
floor from 4.0 to 3.0 is one `redefine`:

```
floor 4.0  →  meets_consensus_floor(AAPL) = False
floor 3.0  →  meets_consensus_floor(AAPL) = True
```

No code edited, no data migrated, every consumer follows. For throughput
you would group the predicates into one query, which TypeDB supports and
this design does not, because it is built for a readable trace.

---

## 5. The policy as a goal tree

The policy is declared as data, in a file that knows nothing about how a
search works:

```
eligible                              AND       cost 1.0
├── coverage_floor                    schema    cost 0.2  [IPS 2.1]
├── quality_bar                       OR        cost 0.5  [IPS 2.2]
│   ├── on_the_mean                   AND       cost 0.5  [IPS 2.2(a)]
│   │   ├── consensus_floor           schema    cost 0.5
│   │   └── not_contested             schema    cost 1.0
│   └── on_conviction                 AND       cost 1.6  [IPS 2.2(b)]
│       ├── strong_conviction         schema    cost 0.8
│       └── broad_coverage            schema    cost 0.9
└── classification_consistent         judgement cost 5.0  [IPS 3.1]
```

Each leaf is tagged by **what is competent to decide it**, and the tag
follows from whether a closed form exists:

- **`schema`** — a threshold over computable numbers. A boolean function
  settles it exactly, for free, with no variance. Five leaves.
- **`judgement`** — a reading of text. One leaf.

### The judgement leaf really has no formula

Worth proving, since a clever query seems plausible. Three securities built
on purpose:

```
NVDA    "Designs GPUs."                industry Technology    consistent
TSLA    (no description)               industry Automotive    absent
WEIRD   "Runs a chain of bakeries."    industry Technology    contradicts
```

The closest writable predicate — does the description contain the sector
name:

```typeql
fun consistent($s: sec) -> boolean:
  match $s has descr $d, has sector $x; $d contains $x;
  return check;

consistent(NVDA)  → False     ← wrong; "Designs GPUs." plainly IS Technology
consistent(TSLA)  → False
consistent(WEIRD) → False
```

`false` for all three, and wrong on the *easy* case. An approximation that
fails the obvious instance cannot be trusted on the ambiguous one.

Note what is **not** the reason. TypeDB expresses "unknown" perfectly well:
`not { $s has descr $d; }` finds the security lacking one, and a
`-> { boolean }` return yields an empty stream for the undecided case. The
model is needed because there is no predicate to write, not because the
schema is short a third truth value.

> Two mistakes from building this, recorded because both were only found by
> running it. An earlier version sent *"is the mean at least 4.0?"* to the
> model, which returned 0.05 where the function returns `false` — slower,
> paid for, less precise. And the first goal tree put `not_contested`
> *outside* the disjunction, making route (b) unreachable: a 5 with spread
> under 2 forces every rating to 4+, so route (a) always clears first. Each
> route now carries its own second condition.

---

## 6. The search

Three tactics, tried in the order given. **The order is the heuristic:**
split, then the free exact answer, then the paid one. Reversing the last two
spends money on arithmetic; nothing prevents it.

| tactic | effect |
|---|---|
| `split` | a compound goal becomes its children (AND or OR) |
| `schema-function` | call a boolean function — closes or fails the goal |
| `ask-model` | ask the decision model — closes, fails, or records a **gap** |

Within a conjunction the cheapest child is tried first, so a failure prunes
the most expensive evidence.

### 6.1 Refuted, with the model never consulted — TSLA

```
[01] search   SPLIT eligible [AND]
[02] search   TRY coverage_floor [IPS 2.1] — schema-function
[03] TypeDB   let $b = meets_coverage_floor($s);   → True
[04] search   CLOSED coverage_floor
[05] search   SPLIT quality_bar [OR — cheapest route first]
[06] search   SPLIT on_the_mean [AND]
[07] search   TRY consensus_floor [IPS 2.2(a)(i)]
[08] TypeDB   let $b = meets_consensus_floor($s);  → False
[09] search   FAILED consensus_floor               ← route (a) dies
[10] search   SPLIT on_conviction [AND]            ← try route (b)
[11] search   TRY strong_conviction [IPS 2.2(b)(i)]
[12] TypeDB   let $b = has_strong_conviction($s);  → False
[13] search   FAILED strong_conviction             ← route (b) dies

TypeDB queries   3 function calls + 0 evidence fetches
model requests   0
verdict          refuted
```

Three things never happened, and that is the result. `not_contested` was
never asked — route (a) needed both halves and the first failed.
`broad_coverage` likewise. And `classification_consistent` was never
reached, so the description and sector were **never fetched** and the model
was **never called**.

The ledger records the absences:

```
coverage_floor : covered by at least two distinct analysts — by schema-function
-- computed by the schema: consensus_floor=false, coverage_floor=true
⊢ not_contested
⊢ on_conviction
⊢ classification_consistent
```

`⊢` means still open. A reviewer can see this verdict is not an opinion
about TSLA's disclosure, because that question was never put.

### 6.2 Backtracking that succeeds — AAPL after a downgrade

Maria Garcia cuts 4 → 1. Ratings 5, 4, 1; mean 3.33.

```
[07-09] route (a) fails on the mean, as above
[10] search   SPLIT on_conviction [AND]
[11] TypeDB   has_strong_conviction($s) → True     (Jane Doe's 5)
[13] search   CLOSED strong_conviction
[14] TypeDB   has_broad_coverage($s)    → True     (three analysts)
[16] search   CLOSED broad_coverage
[17] search   TRY classification_consistent [IPS 3.1] — ask-model
              needs: ['description', 'sector']
[18] TypeDB   has description $v → 'Designs and sells consumer electronics…'
[19] TypeDB   has sector $v      → 'Technology'
[20] Jev      REQUEST  (see §7)
[21] Jev      RESPONSE noul = 0.98
[22] search   CLOSED classification_consistent

TypeDB queries   4 function calls + 2 evidence fetches
model requests   1
verdict          proved
```

Evidence is fetched at step 18, not at the start — until the search reached
that goal there was no reason to pay for it.

The ledger keeps one measurement from the abandoned branch:

```
-- computed by the schema: broad_coverage=true, consensus_floor=false,
                           coverage_floor=true, strong_conviction=true
```

`consensus_floor=false` survives although the proof succeeded by the other
route. The *conclusion* was rolled back by the backtrack; the
*measurement* was not, because it cost a query and is still true. The route
not taken is part of the reasoning.

Commitment is local: having closed `quality_bar` by the second route, the
search does not revisit it if a later goal fails. That is a limitation, not
a completeness guarantee.

---

## 7. The model protocol

The decision model is **stateless** — no stored rules, nothing kept between
calls. Each request carries its own evidence and its own question.

```json
{
  "model": "typesafe/jev-1.13",
  "state": {
    "description": "Designs and sells consumer electronics, software and services.",
    "sector": "Technology"
  },
  "questions": {
    "classification_consistent": {
      "type": "noul",
      "instructions": "Is the issuer's stated business consistent with the sector it is classified under? If the record contains no description of the business, you cannot tell.",
      "criteria": {
        "true": "the description matches the declared sector",
        "false": "the description describes a different industry"
      }
    }
  }
}
```

The `state` is exactly the two keys this goal declares it needs — not the
policy, not the ratings, not the goals already settled. A goal is asked
about its own evidence.

```json
{ "answers": { "classification_consistent": { "type": "noul", "noul": 0.98 } },
  "usage": { "input_tokens": 348, "cost": 0.00001462 } }
```

### One number is enough

A `noul` answer is the probability of yes. `0.98` is near-certain yes,
`0.03` near-certain no, `0.50` is no information. Confidence is derived from
the position rather than self-reported:

```
confidence = 2 × |p − 0.5|      0.98 → 0.96      0.50 → 0.00
```

This is not cosmetic. A model asked *"how confident are you?"* reports
roughly 0.9 regardless — measured at 0.92–0.94 mean across cases including
ones it got wrong. A 0.50 here cannot pretend to be sure, because 0.50 *is*
the uncertainty.

### Three outcomes, and the third is the point

```
confident yes  (p > 0.5, conf ≥ 0.6)  → goal closes
confident no   (p < 0.5, conf ≥ 0.6)  → goal fails, search backtracks
anything else                          → goal closes, a GAP is recorded
```

TSLA has no description. With its arithmetic revised so the search reaches
IPS 3.1:

```json
{ "state": { "description": "(nothing on file)", "sector": "Automotive" } }
→ noul = 0.50
```

*I cannot tell*, which is correct. A binary gate must pick one, and both
choices are wrong: call it no and you sell a holding because a filing has
not loaded; call it yes and you keep one nobody checked. So:

```
verdict         proved (modulo holes)
safe to act on  False
depends on an admitted goal: classification_consistent [IPS 3.1]
```

IPS 2.1 and 2.2 genuinely passed. IPS 3.1 was assumed, the verdict says so
everywhere it is reported, it is not stored, and it goes to a person.

---

## 8. Compared against language models

The obvious objection is *why not just ask a good model?* So it was measured
rather than argued about.

**The benchmark.** 93 cases: every non-decreasing rating profile over 1..5
for one, two and three analysts — the complete space up to ordering —
crossed with three description states. Ground truth is computed from the
policy itself in advance. Reported by stratum, because an aggregate hides
which kind of case a system fails:

```
both_routes_fail 31   unknowable 19   judgement_no 19
route_b_rescue   12   route_a_pass 7  breadth_fail  5
```

`route_b_rescue` is the discriminator: route (a) fails and route (b) must be
tried. `unknowable` is the missing-description set.

### The 93-case result

| | correct | model calls | reasoning tokens | cost |
|---|---|---|---|---|
| **this approach** | **93/93** | **57** | **0** | **$0.0008** |
| claude-opus-5.5, direct | 93/93 | 93 | 6,976 | $0.3316 |
| grok-4.7, step-by-step | 93/93 | 93 | 35,463 | $0.3290 |
| gpt-6.1-sol-pro, step-by-step | 93/93 | 93 | 6,672 | $0.5627 |
| claude-opus-5.5, step-by-step | 93/93 | 92 | 9,369 | $0.9203 |

**100% for every arm, in every stratum** — including all 12
`route_b_rescue` cases and all 19 `unknowable` cases, where every model
answered `review` rather than treating a missing document as a failure.

Repeated three times each on the 31 hardest cases, every arm returned the
same verdict every time. Determinism is a tie too.

So, plainly: **on this task a current frontier model is as accurate and as
consistent as a formal pipeline.** An earlier version of this document
claimed otherwise. That claim was measured against gpt-4o and did not
survive better baselines; it is withdrawn rather than quietly edited out.

### What is left is compute

| | cost | vs. this approach |
|---|---|---|
| this approach | $0.0008 | — |
| grok-4.7 | $0.3290 | 411× |
| claude-opus-5.5, direct | $0.3316 | 415× |
| gpt-6.1-sol-pro | $0.5627 | 703× |
| claude-opus-5.5, step-by-step | $0.9203 | **1,150×** |

Two causes. **57 model calls instead of 93** — thirty-six cases were settled
by arithmetic and never reached a model, and for those the description was
never even fetched. And each of the 57 is one typed yes/no question of 348
input tokens with **no deliberation tokens at all**, against 6,672 to 35,463
for the model arms.

Note also that instructing opus to work step by step **tripled its cost**
($0.33 → $0.92) for identical accuracy. The instruction bought nothing on a
task the model already handled.

At a thousand holdings nightly: roughly $3.60 a night with the cheapest
model, $10 with the step-by-step one, or **under a penny** here.

### The older generation does fail

Measured separately, on three representative cases at ten runs each —
a probe, not the 93-case benchmark, and labelled as such:

| case | correct | gpt-4o, 10 runs |
|---|---|---|
| clear reject | `reject` | reject 8, review 2 |
| route (b) rescue | `include` | review 6, **include 2**, reject 2 |
| missing description | `review` | **reject 9**, review 1 |

Its mean stated confidence across all three was 0.92–0.94 — including on the
case it got wrong eight times in ten. On the third case it rejects a holding
nine times in ten, correctly naming the clause it had no evidence for and
then treating absence as failure.


### The explanation: internal search

Current models emit **reasoning tokens** — an internal deliberation step
before answering. Older ones do not. Captured on the `route_b_rescue` case
(ratings 1, 4, 5), this is opus-5.5's own reasoning:

> *"Three distinct analysts satisfies IPS 2.1. For 2.2, option (a) fails
> since the mean of 3.33 is below threshold and the spread is too wide, **but
> option (b) passes** because Maria Garcia rates it a 5 and three analysts
> cover it. The semiconductor and computing hardware focus aligns with
> Technology under IPS 3.1, so this one should be included."*

Beside the tactic trace on the same case:

| the model's reasoning | the search |
|---|---|
| "Three distinct analysts satisfies 2.1" | ✓ `schema-function on coverage_floor` |
| "option (a) fails … mean of 3.33" | ✗ `schema-function on consensus_floor` |
| "**but option (b) passes**" | ✓ `schema-function on strong_conviction` |
| "and three analysts cover it" | ✓ `schema-function on broad_coverage` |
| "aligns with Technology under 3.1" | ✓ `ask-model on classification_consistent` |

**The same five steps, same order, including the backtrack.** The model
performs the symbolic search internally, in hidden natural language. That is
why the accuracy gap closed: what this system implements as control flow,
recent models have as learned behaviour.

Which reframes what the approach is for. It is not better at the search. It
makes the search **external** — readable, replayable, auditable — and
reaches the same verdicts with **zero deliberation tokens**, because the
steps are database calls rather than generated text.

---

## 9. The judge

The search stores nothing. It returns a plain result, and two checks run.

```
POLICY GATE   a verdict resting on a gap is not asserted
TYPEDB GATE   [CNT8] Constraint '@values("include","review","reject")'
              has been violated: value '"probably-fine"' does not match
              the set of values.
```

That refusal is from the server. `"probably-fine"` is not logged as a
warning — it cannot be stored.

| declared as | refuses |
|---|---|
| `@values("include","review","reject")` | a verdict outside the vocabulary |
| `relates about @card(1)` | a claim about nothing, or about two securities |
| `@range(1..5)` on `rating` | a rating of 7 |
| `coverage plays derived_claim:from_coverage` | a claim grounded in something that is not evidence |
| `@key` on `claim_id` | two claims sharing one identity |

And one the server enforces unprompted:

```
[FRP7] Functions may not contain write stages.
```

A function cannot insert, delete, or nest a `define`. **Reasoning cannot
mutate what it reasons over.**

This separation is what licenses the heuristic. The search may try tactics
in a poor order, go down dead ends, and admit goals — because it is not the
component that decides what counts as known.

---

## 10. What this shows and what it does not

**Shows.** An inspectable record of what is settled, open, and assumed, at
every step and after a refutation. Real backtracking, with a failing route
rolled back and an alternative found. A short-circuit that saves actual
work: zero model calls and zero text fetched when the arithmetic already
refuses. Absent evidence treated as a gap rather than a refusal, with the
resulting verdict marked unsafe to act on. Two independent gates refusing
conclusions the search was content with. And a 400–1,150× compute
difference against models that reach the same answers.

**Does not show.** That the decision model is *necessary* — on this dataset
it changes the verdict only where a description is missing. That the
structure beats a frontier model on *correctness* — it does not; that claim
was made from an older model and is withdrawn. A complete search:
commitment is local, and an alternative that succeeded is never revisited.
Or scale: three securities and eight ratings say nothing about a thousand
holdings.

**Where it would earn more.** This is thresholds over numbers — five of six
goals arithmetic, a domain close to a board game: the state is known, the
moves are defined, the verdict is decisive. The structure matters more where
most goals are the sixth kind — *is this a material change, does this clause
apply* — where no formula exists, missing evidence is the normal case rather
than a contrived one, and the same question changes answer because the rule
changed and not the data.
