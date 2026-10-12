# Does a decision model help the search?

The architecture this example is built to demonstrate:

```
type system   fixes what is ADMISSIBLE — the branches, the prongs,
              and a closed form that settles each one exactly
     ↓
Jev           shown the PROOF STATE, proposes which admissible branch
              to attempt next. Cannot add or remove a branch.
     ↓
search        explores in that order, backtracks on failure
     ↓
database      judges anything written
```

The back half was already real: the search explores, backtracks, and the
database refuses what it will not accept. The front half was not. Branch
order came from constants written by hand, so the model proposed nothing.

This document is what happened when the front half was built and measured.
**It did not pay.** The reason is the useful part.

---

## 1. Why the first example could not show it

The eligibility tree has one OR node with two branches. Measured: a
*perfect* oracle there would save 36 database queries across 93 cases while
costing 88 model calls to ask. The guide costs more than it saves, because
there is almost nothing to guess.

So the domain was widened. A fund runs several mandate sleeves and keeps a
holding if it qualifies for **at least one** — a five-way disjunction of
conjunctions, where picking the wrong branch wastes a whole subtree:

```
retained                                    ALL of
├── coverage_floor                          database
├── a_sleeve_accepts_it                     ANY of
│   ├── index_core        (2 prongs)        database
│   ├── momentum          (2 prongs)        database
│   ├── core_quality      (3 prongs)        database
│   ├── deep_value        (3 prongs)        database
│   └── blue_chip         (3 prongs)        database
└── classification_consistent               Jev
```

14 prongs the database settles exactly; one judgement with no closed form.
Twenty holdings, built so each sleeve wins for some of them and two qualify
under none.

## 2. The seam, and the bound on it

Jev is shown the proof state and the conditions each branch imposes:

```json
{
  "goal_under_consideration": "at least one sleeve's conditions are met",
  "settled": ["coverage_floor"],
  "still_open": ["classification_consistent"],
  "facts_computed_so_far": { "coverage_floor": "true" },
  "admitted_without_proof": []
}
```

and returns an ordering over the five branches.

**It may reorder and nothing else.** `Split` takes the model's *order* and
discards its *objects*, rebuilding the list from the goals the type system
admitted. That distinction is load-bearing, and the first version got it
wrong: it compared names and then installed the proposer's own objects.
`Goal` is a frozen dataclass, so a proposer could return a branch with the
right name and a substituted body that closes true without any of its real
prongs being tested. A generator was worse — it passed the name check,
which consumed it, and then produced an *empty* split that read as closed,
settling a disjunction with no branch proved at all. Both changed the
verdict. Both are refused now, with a test for each.

## 3. The measurement

Twenty holdings, counting database queries. Lower is better.

| ordering | queries | vs. cost |
|---|---|---|
| **schema selection (no model)** | **46** | −64 |
| cost + feature read | 58 | −52 |
| jev, with measures | 67 | −43 |
| perfect OR order, without seeding | 72 | −38 |
| cost ordering (the default) | 110 | — |
| random | 112 | +2 |
| jev, proof state only | 119 | +9 |
| worst (floor) | 141 | +31 |

Two rows decide it.

**`jev, proof state only` — 119, worse than cost ordering.** At the OR node
the proof state holds one boolean, so the model was choosing between five
described branches with nothing to choose on. A policy network without
features is noise.

**`cost + feature read` — 58, against `jev, with measures` — 67.** Both see
the same information. One feature read settles four prongs outright, and
once both orderings have it, **the model's ordering is 9 queries worse than
sorting by a hand-written constant.** The apparent win was never the
ordering; it was the feature read, which any ordering can have.

And `schema selection` — computing which sleeve qualifies directly from the
measures, no model — wins at 46, beating even a perfect branch ordering
that does not seed.

## 4. What this means

**Branch selection in this domain has a closed form.** The five sleeve
predicates are thresholds over numbers the graph already computes, so which
branch will succeed is itself calculable. And anything with a closed form
belongs in the type system — *including the search heuristic*. Putting a
model there is the same mistake as asking it whether a mean exceeds 4.0,
one level up.

That is a sharper statement of where a decision model earns its place than
the original framing allowed:

> A model is justified at a node when **selecting the branch depends on
> something that has no closed form.**

In finance it never does. Every sleeve is arithmetic, so selection is
arithmetic. The architecture is right; this domain cannot exercise its front
half, and widening the disjunction from two branches to five did not change
that — it only made the conclusion measurable.

Where it would differ: a regulatory domain such as the FLSA white-collar
exemptions, a genuine five-way disjunction over Executive / Administrative /
Professional / Computer / Outside Sales, where branches turn on prongs like
*"primary duty is management"* and *"exercises discretion and independent
judgment"* — which have no formula. There you cannot compute which branch
applies, so a proposal is the only way to order them, and a wrong guess
costs the judgement calls of a whole branch rather than a few arithmetic
reads. That port is designed and not built; this document records the
negative result rather than skipping to a domain that would have flattered
it.

## 5. The invariant that held throughout

Every ordering — including the deliberately worst one, and including a
model whose proposals were rejected — **reached the same verdict for every
holding.**

Ordering changes the work. It never changes the answer, because the answer
is the type system's to decide and the database's to accept. That is the
property the whole arrangement exists to provide, and it is the one result
here that did not depend on the domain.
