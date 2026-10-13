# Type Systems and Reasoning

Applying a written rulebook to data, with a **type system deciding every
test that has a closed form** and a decision model asked only the questions
no formula can answer.

The subject is the US white-collar overtime exemption — 29 USC § 213(a) and
29 CFR Part 541. An employee is exempt if they satisfy any one of five
categories: Executive, Administrative, Professional, Computer, or Outside
Sales.

## The split that organises everything

The rulebook has 29 testable parts. Ask one question of each — *could I
write a formula for this?*

**19 of them: yes.**

```
is the salary at least $684 a week?                § 541.600(a)
is the hourly rate at least $27.63?                § 213(a)(17)
does this person direct 2+ full-time equivalents?  § 541.104
is this occupation categorically barred?           § 541.3
does this person hold a qualifying credential?     § 541.301(d)
```

Numbers and lookups. TypeDB functions settle them exactly, for free, with no
variance.

**10 of them: no.**

```
is this person's primary duty MANAGEMENT?                 § 541.100(a)(2)
does this person exercise DISCRETION AND INDEPENDENT
   JUDGMENT on matters of significance?                   § 541.202
is the work DIRECTLY RELATED to management or general
   business operations?                                   § 541.201
does the work require INVENTION, IMAGINATION, ORIGINALITY? § 541.302
are this person's recommendations GIVEN PARTICULAR WEIGHT?  § 541.105
```

There is no column for any of these. § 541.2 is explicit about why: a job
title establishes nothing, **actual duties control** — and duties are prose.
Those ten, and only those ten, go to the decision model.

## What that buys, measured

Run 25 employees through the rulebook twice: once with the decision model,
once with it removed.

```
                 exempt   needs review   not exempt
with the model      14          6              5
without it           0         22              3
```

**Without it, nothing can be cleared.** Every one of the 14 who genuinely
qualify comes back "needs human review", because every exemption branch ends
in a judgement and you cannot reach *exempt* without passing one.

Three determinations survive without it, all **negative** — a line
operative and a paramedic barred by § 541.3, and an employer under the
§ 203(s) revenue floor. That is the asymmetry: a type system can rule people
**out** on its own. It cannot rule anyone **in**.

## The shape

```
the rulebook      a tree of tests, declared as data
                  ↓
the search        split a test into its parts, ask the database,
                  ask the model — in that order
                  ↓            ↓
the database      19 tests      the model      10 tests
                  exact, free                  one number, calibrated
                  ↓
the database      again — the search stores nothing itself
```

Four properties follow:

- **Nothing is ever stale**, because computed values are formulas in the
  database rather than stored numbers.
- **The thresholds live in one place.** Moving $684 is one `redefine`, and
  every caller follows.
- **Absent evidence is not a refusal.** A test nobody can settle is recorded
  as a gap and the verdict says so, rather than being forced to yes or no.
- **The search cannot accept its own conclusion.** A verdict outside the
  permitted vocabulary cannot be stored at all.

## What is here

| | |
|---|---|
| [**EXPLAINED.md**](EXPLAINED.md) | the whole thing: the data model, the 19 functions, the tree, what goes to the model, a worked determination traced call by call, and the measurements |
| [**schema.tql**](schema.tql) | the complete TypeDB schema — 30 attributes, 7 entities, 6 relations, 18 functions |
