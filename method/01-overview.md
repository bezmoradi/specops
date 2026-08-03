# 01 — Overview

## What SpecOps is

A method for writing specifications of system behavior that satisfy two constraints at
once:

- A **human** can read one and tell whether it describes the right behavior.
- An **agent** can execute one against a running deployment and produce a verdict a human
  can trust without re-doing the work.

Most specification formats satisfy the first and not the second. Most test formats satisfy
the second and not the first. The gap between them is where the glue code lives — step
definitions, page objects, bespoke side-effect adapters — and that glue is where the cost
and the rot of end-to-end testing actually accumulate.

An agent with access to the deployed system can eliminate that layer, because it can
figure out *how* to observe something without being told. That is the opportunity. The
risk that comes with it is the subject of the rest of this method.

## Two separations

### Specification, implementation, verification

```
   specification          a human owns it, in version control
        │
        ├──► implementation      an agent writes code to satisfy it
        │
        └──► verification        a different agent proves the deployment matches it
```

The point is that the specification is the only artifact both sides share, and neither
side may edit it to make its own work pass. When an agent writes both the code and the
tests, the two can encode the same misunderstanding and agree with each other perfectly.
A specification written before, and separately from, the implementation is the only thing
that breaks that symmetry.

How well a given suite actually achieves this is a spectrum, not a binary, and it is
measurable. See [`05-provenance.md`](05-provenance.md).

### Acquisition and judgement

```
   the agent ACQUIRES evidence      non-deterministic — and that is the whole value
   the rules JUDGE the evidence     deterministic — and that is non-negotiable
```

An agent asked to *find* the proof that an email was dispatched will do something a
YAML-based test runner cannot: consider the queue, the log, the database row, and the
metric, pick the one that is actually observable in this environment, and go get it.

An agent asked whether the result *counts* as a pass will rationalize. Given a `201
Created` and no evidence of the downstream email, it will reason that the email probably
got sent. This is not an occasional lapse; it is the default behavior of a system
optimized to be helpful.

So the agent's output is never a verdict. Its output is **evidence** — what it ran, what
came back. The verdict is computed from that evidence against the assertions the
specification declared. See [`03-evidence-rules.md`](03-evidence-rules.md) and
[`08-evidence-bundles.md`](08-evidence-bundles.md).

## Threat model

State this before the rules, because several of them are routinely over-read — including
by their author.

**SpecOps defends against a fallible agent. It does not defend against a deceptive one.**

| Failure | Defended? | By what |
|---|---|---|
| Rationalizing an ambiguous result toward green | **Yes** | Verdicts computed from recorded evidence (R1) |
| Treating a missing side effect as probably-fine | **Yes** | Fail-closed (R2) |
| Quietly widening a predicate when a correlator is empty | **Yes** | No-weaker-correlator (R3), and the bundle records the invocation |
| Retrying until the answer is convenient | **Yes** | Bounded polls, counted `INFRA` retries (R4) |
| Skipping an assertion it expected to pass | **Yes** | Every assertion appears in the bundle |
| Reading the *wrong source* and reporting it honestly | **Partially** | Probe targets machine-validated against environment bindings |
| Fabricating a bundle outright | **No** | — |

The last row deserves the attention. A verdict is recomputable from its bundle, and a
fabricated bundle recomputes perfectly — recomputation checks internal **consistency**, not
**truth**. The partial defenses are real but bounded:

- **Binding validation.** A probe invocation can be machine-checked against the environment
  file: a `log(orders)` read that did not use the declared selector is a harness finding,
  not evidence. This catches the wrong-source case, which is the common one.
- **Plan replay.** A cached plan executes a fixed invocation
  ([`11-economics.md`](11-economics.md)), so drift between runs becomes visible.
- **Bundle audit.** A human reading bundles catches what nothing else does — and the method
  is honest that people skip this, which is why the other two matter.

So when [`02-verdicts.md`](02-verdicts.md) says "there is no other path to green," read it
as scoped to the **judgement step**: no reasoning, retry, or adjudication may move a
scenario toward green. It is not a claim that the pipeline is unfoolable.

This is the right threat model for the actual adversary, which is not a malicious model —
it is a helpful one, under time pressure, with an ambiguous result in front of it.

## What kind of testing this is

Being precise about this matters, because the wrong label sets the wrong expectations.

| Axis | Where SpecOps sits |
|---|---|
| **Level** | System / end-to-end, and **grey-box** — the deployed stack is exercised from outside, but evidence is gathered from inside (stores, logs, queues, metrics). |
| **Purpose** | Regression, post-deploy verification, and promotion between environments. |
| **Style** | Specification-based, scenario-driven, behavior-focused. |
| **Oracle** | Hybrid — assertions are explicit and deterministic; evidence acquisition is not. |
| **Execution** | Agent-interpreted. This is the only genuinely new value on any of these axes. |

SpecOps is not a new *kind* of testing. It is a new kind of *runner* for a very old kind
of testing. The closest historical ancestor is specification-by-example — prose documents
with embedded assertions, bound to the system by fixture code. SpecOps removes the fixture
code and lets the agent do that binding at runtime.

Say this plainly when you introduce it to a team. "Agent-executed end-to-end regression
testing with cross-system evidence" is accurate and sets correct expectations. "AI-native
autonomous QA" is not, and invites the team to expect things the method cannot do.

## The three artifacts

A SpecOps suite is not just a folder of specifications. It has three parts, and suites
that fail usually failed by having only the first.

| Artifact | What it holds | Template |
|---|---|---|
| **Specifications** | One per endpoint or per event-driven feature. All scenarios for it, positive and negative. | [`templates/spec.md`](../templates/spec.md) |
| **Execution contract** | How *this* system is verified: what the correlators are, where evidence lives, what the poll budgets are, which shortcuts are permitted and which are forbidden. | [`templates/guide.md`](../templates/guide.md) |
| **Run ledger** | Append-only human record: what ran, against which build, what passed, what was not run and why. | [`templates/RUNLOG.md`](../templates/RUNLOG.md) |

The execution contract is the part people underestimate. Specifications describe *what
should be true*; the contract describes *what counts as having observed it here*. Without
it, every agent invents its own answer to that question and no two runs mean the same
thing.

The run ledger is the part people skip. It is the only artifact that answers "was this
green before the incident?", and no amount of tooling substitutes for a human writing down
what they did and did not run.

## Scope boundary

Say this out loud before adopting, because a method that claims completeness will be
trusted with things it cannot do.

**SpecOps cannot find:**

- **Races and concurrency defects.** A scenario runner drives one actor at a time. A
  genuine multi-writer race needs coordinated concurrent actors and usually a fault
  injector. Pair with an integration harness and accept the gap.
- **Performance regressions.** Latency observed during an agent-driven run is dominated by
  the agent. Pair with a load tool.
- **Input-space coverage.** SpecOps verifies enumerated claims. It cannot tell you about
  the input you did not think of. Pair with a property-based tool driven off your API
  schema.
- **Algorithmic correctness.** If the pricing calculation is wrong in the same way in the
  specification and the implementation, nothing here notices. Only an independently
  derived expectation does.
- **Anything with no observable consequence.** If a behavior leaves no trace in any
  response, store, log, queue, or metric, it cannot be verified. Sometimes the correct
  response is to add the trace.

**What it is unusually good at**, relative to the alternatives:

- Cross-system side effects — the assertion that spans an API call, a queue, a consumer,
  and a database row.
- Negative space — authorization, tenancy isolation, existence oracles, and events that
  must *not* fire.
- Surviving refactors, if the altitude rule is followed ([`04-spec-altitude.md`](04-spec-altitude.md)).
- Being read. A specification that a product owner can review is worth more than a test
  only its author understands.

## Reading order

| Read first | |
|---|---|
| [`02-verdicts.md`](02-verdicts.md) | What a run may conclude, and what it may never conclude. |
| [`03-evidence-rules.md`](03-evidence-rules.md) | The six load-bearing rules. |
| [`04-spec-altitude.md`](04-spec-altitude.md) | How to write a specification that outlives the code. |

| Then | |
|---|---|
| [`05-provenance.md`](05-provenance.md) | Whether your suite is an oracle or a regression net. |
| [`06-side-effects.md`](06-side-effects.md) | Choosing and ranking evidence. |
| [`07-isolation.md`](07-isolation.md) | Running many scenarios without them poisoning each other. |

| When operationalizing | |
|---|---|
| [`08-evidence-bundles.md`](08-evidence-bundles.md) | Making verdicts auditable. |
| [`09-gates.md`](09-gates.md) | What can gate a merge, and what can only gate a deploy. |
| [`10-environments.md`](10-environments.md) | Keeping specifications portable. |
| [`11-economics.md`](11-economics.md) | Making runs affordable and repeatable. |

| Read repeatedly | |
|---|---|
| [`12-false-greens.md`](12-false-greens.md) | The catalogue. Re-read it after every incident a green suite failed to catch. |
