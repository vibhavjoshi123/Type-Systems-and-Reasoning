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
neither side marks its own homework.

| | correct | model calls | reasoning tokens | cost |
|---|---|---|---|---|
| **this approach** | **93/93** | **57** | **0** | **$0.0008** |
| claude-opus-5.5 | 93/93 | 93 | 6,976 | $0.3316 |
| grok-4.7 | 93/93 | 93 | 35,463 | $0.3290 |
| gpt-6.1-sol-pro | 93/93 | 93 | 6,672 | $0.5627 |
| claude-opus-5.5, told to think step by step | 93/93 | 92 | 9,369 | $0.9203 |

**Current frontier models get this task right.** So do we. The difference is
compute: **400 to 1,150× less cost, and no deliberation tokens at all**,
because five of the six questions never reach a model and the sixth is asked
something with a one-number answer.

Earlier-generation models do *not* get it right, and the reason is
instructive rather than embarrassing for them — see
[the comparison](WALKTHROUGH.md#8-compared-against-language-models).

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
