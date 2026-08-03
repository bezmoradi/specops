# 09 — Gates

> An **agent-planned** run needs a deployed system and a model. By the time it runs, the
> code is already live, so it cannot gate a merge.
>
> That is a real limitation, and it is usually accepted as one. It should not be — most of
> what a suite gets wrong is catchable statically, offline, in seconds. And once plans are
> cached, a **replay** run has neither constraint.

**Three tiers, not two.** The distinction that matters is what a run *needs*, not what it
is called:

| Tier | Needs | May gate |
|---|---|---|
| **Static** | Nothing | Every pull request |
| **Replay** (cached plans, no model — [`11-economics.md`](11-economics.md)) | A deployed environment | A deploy, and a merge in trunk-based flows where an environment exists per change |
| **Plan / explore** (agent in the loop) | An environment and a model | A promotion |

So "verification cannot gate a merge" is precise only for the third tier. Replay can gate
anything a conventional API-test suite can, which is the point of caching plans.

## Two gates, two jobs

| | **Static gate** | **Verification run** |
|---|---|---|
| **When** | Pre-merge, on every pull request | Post-deploy, before promotion |
| **Needs** | Nothing. No network, no environment, no model | A deployed system and an agent |
| **Duration** | Seconds | Minutes to hours |
| **Cost** | Free | Real |
| **Catches** | Corpus defects — drift, rot, malformed specifications | Behavior defects — the system does not match the specification |
| **Blocking** | Yes | Yes, for promotion |

Most teams have only the second, and therefore have no feedback at all on the health of
the specification corpus itself until a run fails for a reason that turns out not to be
the system's fault.

## The static gate

Every check below is mechanical, needs no running system, and closes a drift class that
otherwise surfaces only mid-run — as wasted time, or as a false green.

### Structural

| Check | Catches |
|---|---|
| Every specification parses; every typed block is well-formed | A malformed `expect` block that silently evaluates nothing |
| Every scenario has at least one assertion, or an explicit `N/A` declaration | A scenario that can never fail |
| Every `{{VAR}}` is defined before use within its scenario | Empty-capture false greens, before they happen |
| Every scenario id referenced from another specification exists | Stale cross-references after an id is renamed |
| Scenario ids are unique across the corpus | Ambiguous reporting and selection |

### Contract

| Check | Catches |
|---|---|
| Every route in the API schema has at least one specification | Uncovered endpoints, silently |
| Every declared response code appears in the published code manifest | Assertions against invented tokens |
| Every declared rate-limit tier matches the deployed configuration | Silent throttling-contract drift |
| Every declared host or surface exists in the environment bindings | Requests sent to the wrong service — a false green if the other service happens to answer |

### Safety

| Check | Catches |
|---|---|
| Every state-mutating scenario has a `teardown` block | Fixture leakage across runs |
| Every escape hatch is labelled and appears only in setup or teardown | Escape-hatch state being used as evidence of product behavior |
| No probe targets a production binding without an explicit override | The worst possible accident |

### Altitude

| Check | Catches |
|---|---|
| No specification contains a file path, line number, or `.go`/`.ts`/`.py` reference | Over-specification |
| No specification contains an internal type or handler name (heuristic; report, do not block) | Over-specification |
| No assertion depends on a message-copy string where a stable code exists | Specifications that fail on a copy edit |

### Hygiene

| Check | Catches |
|---|---|
| Every assertion carries a provenance tag | Untraceable assertions ([`05-provenance.md`](05-provenance.md)) |
| Every `@intent` tag carries a **resolvable citation** | Provenance inflation — the tag surviving an edit that made the assertion implementation-derived |
| The generated index matches the corpus | An index that has silently stopped describing reality |

### Semantic traps the gate can catch

These are cheap, mechanical, and each maps to a catalogue entry:

| Check | Catches |
|---|---|
| No `count ==`, `still`, or `unchanged` assertion uses `within` **on a subject that predates the stimulus** — mechanically: any `store(…)` count, and any stream count not pinned to a correlator minted in this scenario | Invariants asserted with appearance semantics — [`12`](12-false-greens.md) B6 |
| Every inverted (`absent through`) assertion has a `control` block | Absence scenarios with a dead observation path — [`12`](12-false-greens.md) G1 |
| Every scenario that seeds a subject and asserts unreachability has a seed-visibility control | [`12`](12-false-greens.md) A4 |
| Every `hold` / `exists` / survival assertion declares a consistency path or staleness wait | Stale-read survival — [`12`](12-false-greens.md) B7 |
| Every `within` / `hold` / `absent through` names a budget **class**, never a literal duration | Budgets drifting out of the environment file |
| No assertion targets `.message` where a structured field exists | Message-copy coupling — [`12`](12-false-greens.md) E2 |
| **An exactly-once subject is not asserted by `exists`, nor by `count == n` alone.** Key: a *positive* assertion, on a subject the stimulus produced, pinned to a per-execution-unique correlator, **outside** a `control` block, with no bracketing `hold` / `absent through` anywhere in the scenario | Duplicate delivery — [`12`](12-false-greens.md) B8 |
| Every `deviation` block carries a resolvable citation for its `intent`, and declares both `intent` and `observed` | A `KNOWN-DEVIATION` that is one engineer's opinion ([`02`](02-verdicts.md)) |

**The scope qualifiers on the exactly-once row are load-bearing**, and an earlier draft of it
omitted them. Measured against a real corpus, the unqualified rule — "flag every event
assertion without a cardinality bound" — fired on 17 assertions of which **2** were the
defect: ~12% precision. The other 15 were an inverted assertion already bounded at zero (5)
and liveness probes inside `control` blocks, where `exists` is exactly right and nothing more
is wanted (10). Flagging those is flagging the corpus's best-written scenarios, and the
consequence is two paragraphs down: the gate gets switched off.

Each qualifier removes a specific false positive:

| Qualifier | Excludes |
|---|---|
| *positive* | `absent through` assertions — already cardinality-bounded, at zero |
| *outside a `control` block* | liveness probes, where existence is the whole point |
| *stimulus-produced* | pre-existing subjects — those are B6's territory, and take `hold` |
| *per-execution-unique correlator* | selections that could match another execution's evidence, where a count means nothing anyway |
| *no bracketing wrapper in the scenario* | correctly-written pairs |

The qualifier on the first row is load-bearing, and an earlier version of this table omitted
it. `count == N` is an **invariant** only when the subject already exists as the stimulus is
sent — that is B6's actual mechanism. The same shape used as a **receipt** for something the
stimulus creates, pinned to a correlator minted in this scenario, is an appearance claim and
correctly takes `within`: the dedup-hit assertion in
[`../examples/async-event/order-cancelled.md`](../examples/async-event/order-cancelled.md)
FC2 is exactly this, and the unqualified rule flags it. A gate that fires on a corpus's
best-written scenario gets switched off, which costs more than the check was worth.

## The completeness half

Every check above reads **the assertions you wrote**. None of them can object to the
assertion you *didn't*.

That is not a gap in the list — it is a gap in the method's shape, and it took a trial to
make visible. Two defects escaped a corpus that obeyed every rule on this page: a duplicate
event (the author asserted existence, so no cardinality rule had anything to inspect) and an
integer-overflow boundary (the author probed a large id, but not *the* boundary). Neither is
a violation of anything. Both are omissions, and **a gate that reads the suite is
structurally blind to omissions.**

So this method has two halves, and until recently only shipped one:

| | Question | Source of truth | Status |
|---|---|---|---|
| **Soundness** | Can what you asserted lie to you? | the specification corpus | the checks above |
| **Completeness** | Is there something you must assert and didn't? | **the declared surface** | this section |

The pivot is the source of truth. Soundness checks read the corpus; completeness checks
**must not**, or they inherit exactly the blind spot they exist to find. They read the
system's own declarations — the API schema, the event catalogue, the column types, the error
manifest — and *generate obligations*, which the corpus then either satisfies or fails.

Three that are cheap and mechanical today:

| Declared source | Generates | Would have caught |
|---|---|---|
| **Event / topic catalogue**, with each event's multiplicity | For every exactly-once event: an obligation that some scenario asserts it with a receipt **and** a bracket | the duplicate-publish miss |
| **Column and parameter types** (schema, OpenAPI) | For every integer-typed path or query parameter: probes at `max`, `max+1`, `min−1`, and the width boundary of the *storage* type where it is narrower than the parsing type | the `int4`/`int64` overflow — `int4` yields `2147483648` on the first try, where judgment produced `2147483000` and `999999999` |
| **Response-code manifest** | For every declared code on every route: an obligation that some scenario elicits it | declared-but-unreachable error paths |

Note what the second row really says. All three suites in the trial picked boundary values by
**judgment**, and two of three picked values on the safe side of a boundary they were
explicitly hunting for. Judgment is not the reliable input here; the declared type is. The
storage type mattering more than the parsing type is the specific trap — code that validates
with a 64-bit parse and stores in a 32-bit column has a live window between them, and only
the *narrower* declaration reveals it.

**A completeness check reports a corpus finding, not a verdict.** It says "nothing asserts
X," which is a statement about the suite, and the response is to write the scenario. It must
never be able to turn a run red on its own — that would make a coverage number blocking,
which this page rejects two sections down for good reasons that still apply.

**Where the source of truth does not exist, say so rather than substituting judgment.** A
project with no event catalogue cannot generate exactly-once obligations, and the honest
output is "this check could not run," not a quieter version of the same gap. That absence is
worth surfacing: it usually means nobody has written down which events are exactly-once,
which is a more serious problem than any individual missing assertion.

### Self-conformance

**The corpus of this method's own templates and examples must pass this gate.** A
methodology whose examples fail its own checks is teaching the violations, and the examples
are the artifacts people copy. Run the gate over `templates/` and `examples/` in the
project's own CI.

**The index should be generated, not hand-maintained.** A hand-written catalogue of
hundreds of specifications drifts within weeks, and drifts in the direction of claiming
coverage that does not exist.

## Ordering: cheap gates first

Run the static gate before the verification run, always. A run that fails after forty
minutes because a specification referenced an undefined variable has wasted forty minutes
and an environment.

```
pull request      →  static gate                    (seconds, free)
merge + deploy    →  smoke subset                   (minutes)
promotion         →  full verification run          (the confidence gate)
```

## Selection: which scenarios to run

**The full suite is the only confidence gate.** For promotion, run everything. Any subset
is an argument that the parts you skipped could not have been affected, and that argument
is wrong more often than it is right.

For a quick check of one area, select **coarsely by tag**. Coarse selection fails toward
over-running, which is safe. It can never silently under-select, which is the property that
matters.

### On change-driven selection

The appealing idea is to compute "the exact scenarios this diff could have affected" and
run only those. It has been tried, and the usual approach — maintaining a map from source
symbols to scenarios — fails the same way every time: the map is hand-maintained, it rots,
and a rotted selection map does not announce itself. It silently under-selects, which is
the one failure mode selection must not have.

If you want change-driven selection, **derive it from execution, not from a map**:

1. Bundles record the trace identifiers each scenario produced
   ([`08-evidence-bundles.md`](08-evidence-bundles.md)).
2. Your tracing infrastructure knows which code served those traces.
3. Therefore "scenarios that exercised the changed files, last run" is a query.

Self-maintaining, because it is recomputed from every run. Nothing to keep in sync. If you
do not have the tracing to support it, stay with coarse tags — do **not** hand-maintain a
map.

## The conformance kit

The static gate checks the corpus. Nothing so far checks the **evaluator** — and under R1
the evaluator is what decides every verdict, so an evaluator that is subtly wrong is a
silent, total failure.

You do not need to ship an implementation to make evaluators testable. Publish a
**conformance kit**: a specification of evaluator behavior plus a set of golden triples,
as data.

```
conformance/
├── EVALUATOR.md                        normative: how each assertion form evaluates
└── cases/
    ├── 001-within-appearance/          spec fragment · evidence bundle · expected verdict
    ├── 002-hold-invariant/             the FC2 class: hold vs within on a count
    ├── 003-empty-vs-errored/           FAIL vs INFRA classification
    ├── 004-empty-capture/              unresolved correlator → FAIL, never a fallback
    ├── 005-absent-through-no-control/  inverted verdict with and without a control
    ├── 006-blocked-computed/           missing requirement → BLOCKED, not judged
    ├── 007-no-short-circuit/           a missing assertion is not a passing one
    ├── 008-hold-missing-final-sample/  a whole-budget claim needs a sample at expiry
    ├── 009-exactly-once/               "at least one" cannot satisfy "exactly one"
    └── 010-known-deviation/            three-way resolution of a declared deviation
```

Each case is `(spec, bundle, expected_verdict)`. Any evaluator — a program, or an agent
following the prompts — can be run against them, and disagreement is a bug in the evaluator
rather than a matter of opinion.

This is the cheapest available answer to "the method says the agent must not vote, but
provides nothing that stops it." It does not stop it. It makes an adopter able to **audit**
their executor in an afternoon, and it gives any future runner a definition of correct.

## Not everything belongs in a gate

Two things to resist:

**A coverage percentage as a blocking threshold.** It measures enumeration, not
verification quality. A suite can assert `status == 200` on every endpoint and report total
coverage while proving nothing. The provenance ratio ([`05-provenance.md`](05-provenance.md))
is a more honest number and still should not be a blocking threshold.

**Blocking on `INFRA`.** If the environment is broken, the correct outcome is a loud
`INFRA` report and a retry, not a blocked merge that trains people to bypass the gate.
Distinguish "your change is bad" from "our environment is down," and only block on the
first.
