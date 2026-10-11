# Type Systems and Reasoning

A worked example of applying a written policy to data with a **type system
doing the deciding wherever a formula exists**, and a language model asked
only the questions no formula can answer.

The domain is portfolio eligibility: a fund's written rule about which
holdings may stay. It has six testable parts. Five are arithmetic over
numbers the database can compute. One is a reading of English. That split —
and what follows from it — is the whole subject.

## The result, up front

93 test cases, correct answers derived from the rule itself in advance, so
neither side marks its own homework. Nine arms:

- **This approach — 100%**, 57 model calls, **zero** reasoning tokens, **$0.0008**
- **Four current frontier models — 100% each**, 93 calls, 6,811–36,884 reasoning tokens, $0.34–$0.93
- **gpt-4o answering directly — 72.6%**, failing at backtracking (33%) and missing evidence (24%)
- **gpt-4o told to work step by step — 100%**

That last pair is the finding. **The same model, 72.6% answering directly
and 100% when made to work through the clauses.** Nothing about its weights
changed — writing the search down is what fixed it.

So the capability is the *search*, not the model. Recent models run it
internally, which is why they need no instruction and why they match us on
accuracy. What differs is 400–1,150× in compute, and whether the reasoning
is visible.

## What is in here

| | |
|---|---|
| [**EXPLAINED.md**](EXPLAINED.md) | the whole thing from scratch, assuming no familiarity with type systems or theorem provers. Start here. |
| [**WALKTHROUGH.md**](WALKTHROUGH.md) | the technical account: the schema, the function dependency graph, the search, the model protocol, the model comparison. |
| [**schema.tql**](schema.tql) | the TypeDB additions — 11 provenance attributes, 4 relation types, 14 functions. Pure `define` over a published base model. |

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

Four properties follow, and each is the subject of a section:

- **Nothing is ever stale**, because computed values are formulas in the
  database rather than stored numbers.
- **The policy lives in one place.** Moving a threshold is one line, and
  every consumer follows.
- **Absent evidence is not a refusal.** A question nobody can settle is
  recorded as a gap, and the verdict says so.
- **The reasoning cannot accept its own conclusion.** A verdict outside the
  permitted vocabulary cannot be stored at all.

## Credit and scope

The base data model — securities, analysts, coverage relations, price bars,
indices — is TypeDB's published `agentic-finance` example, used unmodified.
Everything in `schema.tql` is applied on top as pure `define`.

The decision model is Jev, reached over OpenRouter's System One endpoint. It
is stateless: it stores no rules and keeps nothing between calls.

This is one worked example, not a library. It is deliberately a domain of
thresholds over numbers, which is the easy case — the closing section of the
walkthrough says where that flatters the approach and where it does not.
