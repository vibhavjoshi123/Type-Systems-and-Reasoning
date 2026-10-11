# Explained from scratch

Same material as the [walkthrough](WALKTHROUGH.md), assuming no familiarity
with type systems or theorem provers. Every term is defined where it first
appears; there is a [glossary](#glossary) at the end.

---

## 1. The job

A fund owns shares. Someone wrote a rule for which shares may stay. It has
three parts, and **all three** must be true:

1. **At least two analysts** cover the stock.
2. It **clears a quality bar** — two different ways, either one is enough:
   - **Way A**: average rating 4.0 or better, *and* the analysts mostly agree.
   - **Way B**: at least one analyst gives it 5 out of 5, *and* at least
     three analysts follow it.
3. The company's **written description of its business** matches the
   **industry** it is filed under.

Apply that to every holding, every night, and be able to answer three
questions the next morning: what was decided and why, what was *not* known,
and whether the chain of reasoning can be trusted.

---

## 2. The data

Analysts rate stocks 1 to 5. Eight ratings exist:

| | Jane Doe | John Smith | Maria Garcia | average | highest − lowest |
|---|---|---|---|---|---|
| **AAPL** | 5 | 4 | 4 | 4.33 | 1 |
| **NVDA** | 5 | 5 | — | 5.00 | 0 |
| **TSLA** | 3 | 2 | 4 | 3.00 | 2 |

Each company has a one-line business description — **except TSLA, which has
none.** That gap turns out to be the most interesting case here.

A rating is stored on **the act of one analyst covering one stock**, not on
the analyst and not on the stock. The reason: a verdict later needs to point
back at the specific ratings behind it. You can point at a record. You
cannot point at a number flattened into a column with twenty others.

---

## 3. What a database function is

This is the central idea. Here it is as a spreadsheet.

You want AAPL's average rating in a cell. Two ways:

- **Type the answer.** Work out 5 + 4 + 4 = 13, ÷ 3 = 4.33, type `4.33`.
- **Type a formula.** Type `=AVERAGE(B2:B4)`. The cell shows 4.33.

On screen, identical. Then Maria Garcia changes her 4 to a 1:

- The typed number still reads `4.33`. **It is now wrong**, until a human
  remembers to retype it.
- The formula reads `3.33` immediately. Nobody did anything.

**A TypeDB function is the formula.** It lives in the database and is worked
out fresh every time it is asked:

```typeql
fun consensus_rating($s: security) -> double:
  match
    $cov isa coverage, links (covered: $s, analyst: $a);
    $cov has rating $r;
  return mean($r);
```

*Given a stock, find every coverage record about it, take the rating off
each, return the mean.* `-> double` means the answer is a decimal.

Proof from a real run:

```
before   consensus = 4.3333     meets_floor = True
         ← one coverage record rewritten. No "recalculate" command.
after    consensus = 3.3333     meets_floor = False
```

**No stale number, because no stored number.**

### A function that answers yes or no

The rule asks *is the average 4.0 or better?* So:

```typeql
fun meets_consensus_floor($s: security) -> boolean:
  match
    let $r in consensus_rating($s);      ← this function calls the other one
    $r >= 4.0;                           ← the policy's threshold
  return check;                          ← true if that was satisfiable
```

`-> boolean` means `true` or `false`. Not a score, not a likelihood.

### Formulas on formulas

Spreadsheet formulas can use cells containing other formulas. So can these.
The deepest chain is four levels:

```
index_weight          "what share of the Nasdaq 100 is AAPL?"
  needs market_cap            "AAPL's total value"
          needs latest_price       "AAPL's most recent price"
  needs index_market_cap      "the index's total value"
          needs market_cap         "…for every constituent"
                  needs latest_price
```

I inserted **one new price, for NVDA**:

```
AAPL weight   77.82%  →  53.38%
```

Nothing about AAPL changed. The index grew, so AAPL is a smaller share of
it — and the right answer appeared four levels up with nothing told to
update. With stored numbers you would need a system that knows *"a new price
for any company changes every other company's index weight"*. That system is
where bugs live. Here it need not exist.

### Why the rule goes in the database too

`meets_consensus_floor` contains **4.0**. That is the policy, not data about
companies. I changed it to 3.0 — one line, in the database:

```
floor 4.0  →  AAPL does not meet it
floor 3.0  →  AAPL meets it
```

**No program edited, no data migrated.** Everything asking the question gets
the new answer, because there is one definition of it. Compute the average
in your own program instead and the policy exists in two places, and one day
they disagree.

**The cost, honestly:** a function call takes 251.7 ms; a raw ratings fetch
takes 254.8 ms. So functions lose on *round trips* — three to five calls per
stock where a program could fetch once and compute locally. What you buy is
the single definition.

---

## 4. Who can answer each question

Take the rule and ask of each piece: **could I write a formula for this?**

| question | formula? | why |
|---|---|---|
| at least two analysts? | **yes** | count the records |
| average 4.0 or better? | **yes** | add, divide, compare |
| do the analysts agree? | **yes** | highest − lowest, under 2? |
| anyone rated it 5? | **yes** | is 5 among them |
| at least three analysts? | **yes** | count again |
| does *"Designs GPUs and accelerated-computing platforms"* match *"Technology"*? | **no** | — |

Five yes, one no. The five are arithmetic: a formula answers them exactly,
instantly, for nothing. The sixth is not — there is no sum to perform on
those words.

### Proof that the sixth cannot be written

Three companies, built on purpose:

```
NVDA    "Designs GPUs."                industry Technology    consistent
TSLA    (no description)               industry Automotive    absent
WEIRD   "Runs a chain of bakeries."    industry Technology    contradicts
```

The closest formula anyone can write — *does the description contain the
industry name?*

```typeql
consistent(NVDA)  → False     ← wrong. "Designs GPUs." plainly IS Technology.
consistent(TSLA)  → False
consistent(WEIRD) → False
```

`false` for all three, and it gets the **easy** case wrong: the word
"Technology" is simply not in NVDA's sentence. A formula that fails the
obvious case cannot be trusted on the ambiguous one.

So the sixth question, **and only the sixth**, goes to a language model.
That is the entire division of labour, and it is forced by the data rather
than chosen.

> An earlier version of this sent *"is the average at least 4.0?"* to the
> model, which answered **0.05** where the database returns `false`. Slower,
> paid for, and less precise. Asking a model to do arithmetic is not
> neutral; it is strictly worse.

---

## 5. The search

The **search** walks the checklist, under three rules:

1. **Ask the cheap questions first.**
2. **Stop as soon as the answer is settled.**
3. **Write down every step, including the failures.**

It has three ways of trying to settle a question, called **tactics**, and
the order matters:

| tactic | what it does |
|---|---|
| **split** | a question made of smaller questions becomes those questions |
| **ask the database** | a yes/no function — free, exact |
| **ask the model** | costs money, returns a probability |

**The order is the judgement.** Split, then the free answer, then the paid
one. Reverse the last two and you pay a model to do arithmetic.

The checklist is a tree, because the rule has "all of" and "either/or" parts:

```
is this holding eligible?                 ALL of the below
├── at least two analysts                 database
├── clears the quality bar                EITHER of
│   ├── Way A                               ALL of
│   │   ├── average 4.0 or better          database
│   │   └── analysts not split             database
│   └── Way B                               ALL of
│       ├── someone rated it 5             database
│       └── at least three analysts        database
└── description matches the industry       model
```

> A mistake found by running it: the first version put "analysts not split"
> *outside* the either/or, as a fourth top-level requirement. That made Way B
> unreachable — a 5 with a spread under 2 forces every rating to 4 or more,
> so the average always clears 4.0 and Way A always wins first. Each way now
> carries its own second condition.

---

## 6. Two stocks, walked through

Real logged output. Each numbered line is one thing that happened.

### TSLA — refused, and the model is never consulted

Rated 3, 2, 4. Average 3.00, nobody at 5.

```
[01] search   SPLIT eligible [ALL]
[02] search   TRY at least two analysts
[03] TypeDB   meets_coverage_floor(TSLA)  → True
[04] search   SETTLED at least two analysts
[05] search   SPLIT quality bar [EITHER — cheapest way first]
[06] search   SPLIT Way A [ALL]
[07] search   TRY average 4.0 or better
[08] TypeDB   meets_consensus_floor(TSLA) → False
[09] search   FAILED average 4.0 or better      ← Way A is dead
[10] search   SPLIT Way B [ALL]                 ← try the other way
[11] search   TRY someone rated it 5
[12] TypeDB   has_strong_conviction(TSLA) → False
[13] search   FAILED someone rated it 5         ← Way B is dead too

database queries   3     text fetched  none     model calls  0
verdict            refused
```

**Three things never happened, and that is the result.**

- *"Are the analysts split?"* never asked — Way A needed both halves and the
  first failed.
- *"At least three analysts?"* never asked, same reason in Way B.
- *"Does the description match?"* never reached — so the description was
  **never fetched** and the model was **never called**.

The record shows the absences:

```
at least two analysts : settled
⊢ analysts not split            ← never asked
⊢ Way B                         ← never asked
⊢ description matches           ← never asked
```

`⊢` means still open. Anyone reviewing this knows the verdict is not an
opinion about TSLA's description, because that was never put to anything.

### AAPL after a downgrade — the search finds the other way

Maria Garcia cuts 4 → 1. Ratings 5, 4, 1; average 3.33. Steps [01]–[09]
identical: two analysts ✓, Way A fails on the average. Then:

```
[10] search   SPLIT Way B [ALL]
[12] TypeDB   has_strong_conviction(AAPL) → True    (Jane Doe's 5)
[15] TypeDB   has_broad_coverage(AAPL)    → True    (three analysts)
[16] search   SETTLED the quality bar — by the other way
[17] search   TRY description matches — ask the model
              needs: description, sector
[18] TypeDB   description → 'Designs and sells consumer electronics…'
[19] TypeDB   sector      → 'Technology'
[20] model    asked
[21] model    0.98
[22] search   SETTLED description matches

database queries  4 + 2 fetches    model calls  1    verdict  keep
```

Going back and trying the second way after the first fails is called
**backtracking**. It sounds obvious written down; it is the thing most
easily got wrong, because something has to *remember* there was a second way
after the first already failed.

Note the information is fetched at step 18, not at the start. Until the
search reached that question there was no reason to pay for it. In the TSLA
case it never happened.

---

## 7. What the model is sent

Not the policy, not the ratings, not what is already decided. Just the two
facts this one question needs:

```json
{
  "state": {
    "description": "Designs and sells consumer electronics, software and services.",
    "sector": "Technology"
  },
  "questions": {
    "classification_consistent": {
      "type": "noul",
      "instructions": "Is the issuer's stated business consistent with the sector it is classified under? If the record contains no description of the business, you cannot tell."
    }
  }
}
```

It replies with **one number**:

```json
{ "classification_consistent": { "noul": 0.98 } }
```

```
0.98   almost certainly yes
0.50   no idea at all
0.03   almost certainly no
```

One number is enough because the uncertainty is in its *position*:

```
confidence = 2 × |number − 0.5|       0.98 → 0.96        0.50 → 0.00
```

That matters. Ask a model *"how confident are you?"* and it will say about
0.9 whatever the truth — measured at 0.92–0.94 across cases including ones
it got wrong. Here a 0.50 cannot pretend to be sure, because 0.50 **is** the
uncertainty.

### Three outcomes, not two

```
confident yes   →  the question is settled yes
confident no    →  settled no; the search backtracks
anything else   →  settled by assumption, and a GAP is recorded
```

---

## 8. The missing description

TSLA has no description. With its ratings revised so the first two
requirements pass, the search reaches the third and asks:

```json
{ "state": { "description": "(nothing on file)", "sector": "Automotive" } }
→ 0.50
```

*There is nothing here to compare; I cannot tell.* Correct.

A plain yes/no system must pick one, and **both are wrong**:

- Call it **no** → you sell a holding because a document was not uploaded.
- Call it **yes** → you keep a holding nobody checked.

So neither. The verdict is marked as **resting on a gap**:

```
verdict         proved (modulo holes)
safe to act on  False
depends on an assumed question: description matches the industry
```

Requirements 1 and 2 genuinely passed. Requirement 3 was assumed, that is
recorded everywhere the verdict is reported, it is not saved, and it goes to
a person.

A correction worth keeping: an earlier version of this document said the
database *cannot express* "unknown". That was wrong. `not { $s has descr
$d; }` finds the stock lacking one, and a function returning a list can come
back empty for the undecided case. The real reason is §4 — **there is no
formula for the question at all.**

---

## 9. Why the database has the last word

The part that works all this out **cannot save anything**. It returns a
result, and two checks run.

**Is the verdict resting on a gap?** If so it is not saved.

**Does the database accept it?** The verdict field is declared as exactly
three permitted words:

```
[CNT8] Constraint '@values("include", "review", "reject")' has been
       violated: value '"probably-fine"' does not match the set of values.
```

`"probably-fine"` is not logged as a warning. **It cannot exist.**

The database also refuses a rating of 7 (`@range(1..5)`), a verdict about no
stock or two stocks (`@card(1)`), two verdicts sharing an identity
(`@key`), and a verdict "based on" something that is not a rating. And on
its own initiative:

```
[FRP7] Functions may not contain write stages.
```

A function cannot insert or delete. **Reasoning cannot change the thing it
is reasoning about.**

This separation is what makes the rest safe. The search may try things in a
poor order, go down dead ends, and be unsure — because it is not the
component that decides what is true.

---

## 10. Compared against language models

93 test cases, correct answers worked out in advance from the rule itself.

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


### Why the recent models succeed

They emit **reasoning tokens** — an internal deliberation step before
answering. Older models do not. Here is what one of them actually thought on
the hard case (ratings 1, 4, 5 — Way A fails, Way B rescues):

> *"Three distinct analysts satisfies IPS 2.1. For 2.2, option (a) fails
> since the mean of 3.33 is below threshold and the spread is too wide, **but
> option (b) passes** because Maria Garcia rates it a 5 and three analysts
> cover it. The semiconductor and computing hardware focus aligns with
> Technology under IPS 3.1, so this one should be included."*

Against our own steps on the same case:

| what the model thought | what the search did |
|---|---|
| "three analysts satisfies 2.1" | ✓ asked the database |
| "option (a) fails … mean of 3.33" | ✗ asked the database |
| "**but option (b) passes**" | ✓ asked the database |
| "and three analysts cover it" | ✓ asked the database |
| "aligns with Technology" | ✓ asked the model |

**The same five steps, in the same order, including the backtrack.** The
model does the search internally, in hidden prose. That is the whole reason
the accuracy gap closed: what this system implements as explicit control
flow, recent models have as learned behaviour.

Which says what the approach is actually for. It is not better at the
search. It makes the search **external** — readable, replayable, auditable —
and reaches the same verdicts with **no deliberation tokens at all**,
because the steps are database calls rather than generated text.

---

## 11. Common confusions

**Is the model deciding anything important?** One of six questions. It
changes the verdict only where a description is missing. Narrow by design.

**Why not put all the rules in the database and drop the model?** You can,
for five of six. The sixth has no formula — §4.

**Why not ask a model everything?** On accuracy you could. It costs
400–1,150× more, you cannot see which ratings an answer used, and nothing
stops it returning a verdict outside the three permitted words.

**Isn't the record just logging?** Logging is written alongside the work.
This record *is* the work — appended to as each step runs, and nothing can
be in the verdict that is not in it.

**Why keep failed steps?** The way not taken is part of the reasoning. A
record showing only what worked is a story written after the answer.

**What if the database is wrong?** Then the verdict is wrong — and the
record shows exactly which ratings it relied on, so you can tell. That is
why they are attached.

---

## Glossary

| term | meaning |
|---|---|
| **function** | a formula stored in the database, worked out fresh each time. Never stale, because never stored. |
| **relation** | a record joining things together that can carry its own data — `coverage` joins an analyst to a stock and carries the rating. |
| **`@card(1)`** | "exactly one of these" — a constraint the database enforces. |
| **`@values(...)`** | "only these exact values are permitted". |
| **goal** | one question that must be settled. |
| **tactic** | a way of trying to settle one: split it, ask the database, ask the model. |
| **ALL / EITHER** | every part must hold / any one part suffices. |
| **backtracking** | after one route fails, undoing it and trying the next. |
| **short-circuiting** | not asking a question whose answer cannot change the outcome. |
| **ledger** | the running record: settled, still open, assumed. |
| **`⊢`** | "still to be settled". |
| **gap** | a question closed by assumption because nobody could tell. |
| **noul** | a yes/no question whose answer is one number: the probability of yes. |
| **reasoning tokens** | a model's internal deliberation before it answers. Recent models have them; older ones do not. |
