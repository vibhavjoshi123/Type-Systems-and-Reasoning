# Type Systems and Reasoning

## The shape of it

```
the rule          a tree of obligations, declared as data
                  ↓
the procedure     tactics: split a goal, ask the database, ask the model
                  ↓           ↓
the database      exact answers        the model     judgements
(5 of 6 goals)    free, no variance    (1 of 6)      one number, calibrated
                  ↓
the judge         the database again — the procedure cannot store anything
```

## The case: portfolio eligibility

A fund's written policy, six testable parts. Asking one question of each —
*could I write a formula for this?* — splits them cleanly:

- **Five are arithmetic.** Count the analysts, average the ratings, measure
  the spread. TypeDB functions answer these exactly, for free, with no
  variance.
- **One is a reading of English**: does a company's stated business match
  its filed sector? No formula exists. The closest writable one answers
  `false` for a company described as *"Designs GPUs"* filed under
  Technology — it fails the easy case.

That one question, and only that one, goes to Jev.

## Search, provenance, verification

- Obligations form a tree with AND and OR nodes. The search tries the cheap
  route first and **backtracks** when it fails — the `first | t₁ | t₂`
  combinator, as control flow rather than instruction.
- Functions are computed on demand, never stored. One new price bar moved
  one holding's index weight from 77.82% to 53.38%, four levels up the call
  chain, with nothing told to update. **Nothing can go stale because nothing
  is cached.**
- **The policy lives in one place.** The 4.0 threshold is a line in a
  database function, not a constant in a program. Moving it is one
  `redefine`, and every consumer follows — the nightly job, an ad-hoc
  query, a dashboard.
- A verdict links the specific coverage relations and price bars it was
  computed from **as role players, not copied values**. "Why?" is a graph
  walk.
- When evidence is absent, Jev returns 0.50 — *I cannot tell* — and the
  verdict is marked as resting on a gap. Not a refusal. A binary gate must
  guess, and both guesses are wrong.
- The search **cannot store its own conclusion**. A verdict outside the
  three permitted values is refused by the database:
  `[CNT8] value '"probably-fine"' does not match the set of values`.

---

[EXPLAINED.md](EXPLAINED.md) — from scratch ·
[WALKTHROUGH.md](WALKTHROUGH.md) — the technical account ·
[schema.tql](schema.tql) — the TypeDB additions
