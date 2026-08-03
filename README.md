# SpecOps

**A method for writing system specifications that an AI agent can verify against a
running deployment — and whose verdicts a human can actually trust.**

SpecOps is not a tool. There is nothing to install. It is a set of rules, templates, and
prompts you copy into your own repository so that your coding agent writes, and later
verifies, specifications for *your* system.

---

## The problem

The default AI workflow has a hole in it:

```
requirement ──► agent writes the implementation
requirement ──► agent writes the tests
                └─► everything passes
```

The implementation and the tests can encode the **same misunderstanding** of the
requirement. When they do, a green suite tells you nothing. Tests written by the author
of the code — human or model — are not an independent oracle.

There is a second, subtler hole. When an agent both *executes* a test and *decides
whether it passed*, it will rationalize. Asked "did this pass?", a model presented with a
`201 Created` and a missing downstream email will reason that the email "probably" got
sent. That is not a hypothetical failure mode; it is the dominant one.

## The method

SpecOps separates three things that normally collapse into one:

```
   specification          a human owns it, in Markdown, in git
        │
        ├──► implementation      an agent writes code from it
        │
        └──► verification        a different agent proves the deployment matches it
```

And it separates two more:

```
   the agent ACQUIRES evidence        ← non-deterministic, and that is fine
   the rules JUDGE the evidence       ← deterministic, and that is mandatory
```

The agent is very good at figuring out *how* to observe something — whether the proof
that an email was dispatched lives in a queue, a log line, a database row, or a metric.
It is very bad at being asked whether the result is good enough. So it is never asked.

Six rules carry the weight. They are stated in full in
[`method/03-evidence-rules.md`](method/03-evidence-rules.md):

1. **The agent never votes.** A verdict is computed from recorded evidence against a
   declared assertion. No reasoning, retry, or adjudication may move a scenario *toward*
   green. (Scoped precisely: this is a claim about the judgement step, not a claim that the
   pipeline cannot be fooled — see the [threat model](method/01-overview.md#threat-model).)
2. **Fail closed.** Missing, empty, or errored evidence is `FAIL` — never
   `PASS`-by-inference. A `2xx` response never excuses an absent side effect.
3. **No weaker correlator.** If an assertion depends on a correlator and the correlator is
   empty, that is a `FAIL` — not permission to fall back to a broader match that would
   alias someone else's evidence into your green.
4. **Bounded poll, never re-run-until-green.** Async evidence is polled to a declared
   budget. Absent after the budget is `FAIL`. Re-running until it passes is precisely how
   a real regression walks through a fail-closed gate.
5. **Write at an altitude that survives refactoring.** A specification states observable
   behavior. It never names a class, a handler, a file, or a line number. A
   behavior-preserving refactor must leave every specification valid.
6. **Every assertion declares its provenance.** An assertion derived from reading the
   implementation is a regression check. An assertion derived from a requirement is an
   oracle. Both are useful; conflating them is how a suite comes to believe it is
   something it is not.

## What is in this repository

| Directory | What it is |
|---|---|
| [**`method/00-summary.md`**](method/00-summary.md) | **The whole method in one page.** Start here. |
| [`method/`](method/) | The normative content. Fourteen numbered, citable pages. |
| [`templates/`](templates/) | Files you copy into your own repo: the spec skeleton, the execution contract, the environment bindings, the run ledger. |
| [`prompts/`](prompts/) | What to actually say to your coding agent — to author specs, to execute them, to review them. |
| [`examples/`](examples/) | Worked specifications, a filled-in execution contract, and a real evidence bundle. |
| [`conformance/`](conformance/) | How an evaluator must behave, plus golden `(spec, bundle, verdict)` cases so you can audit yours. |
| [`adoption/`](adoption/) | Day one to first green scenario, and a checklist for whether you are doing this correctly. |

If you read one page after the summary, read
[**`method/12-false-greens.md`**](method/12-false-greens.md) — the catalogue of **34 ways**
an agent-executed suite reports green while the system is broken. It is the part of this
method that was learned the expensive way, and it is the part most likely to change how you
write your next test.

## Provenance, and an honest note about it

This method was extracted from a production suite of roughly **720 scenarios across three
services**, run against a live multi-service cloud deployment.

**What it has demonstrated, stated precisely.** Used as the regression gate for a full
restructuring of one service's internal architecture — every package moved, no observable
behavior changed — it returned **563 PASS / 0 FAIL**, with every non-passing scenario a
pre-documented fixture gap rather than a discovered regression.

That is evidence for **one** rule: altitude. Specifications written at the right altitude
survive refactoring, and that claim is now tested rather than asserted.

It is **not** evidence that the suite is hard to fool. A behavior-preserving refactor cannot,
by construction, say anything about false-green resistance — and by this method's own
epistemology a zero-failure run is ambiguous between "high altitude" and "cannot fail"
([`adoption/checklist.md`](adoption/checklist.md) asks exactly that question). The honest
headline for any suite is **how many real defects it has caught**. If you adopt this, that is
the number to track and publish.

It is also, deliberately, a record of what that suite got **wrong**. Five pages here
describe practices that suite does not yet follow, and which it needs:

| Page | The gap it closes |
|---|---|
| [`05-provenance.md`](method/05-provenance.md) | Specs derived by reading the implementation are a regression net, not an independent oracle. Most suites are honest about neither. |
| [`08-evidence-bundles.md`](method/08-evidence-bundles.md) | A suite that argues "evidence over assertion" and then archives nothing is not auditable. A past PASS must be re-examinable. |
| [`09-gates.md`](method/09-gates.md) | Agent-executed verification runs post-deploy and cannot gate a merge. A static gate over the corpus can, costs seconds, and catches most real drift. |
| [`10-environments.md`](method/10-environments.md) | Specs with hostnames, regions, and cluster commands inlined are bound to one deployment forever. |
| [`11-economics.md`](method/11-economics.md) | An agent in the loop on every run is slow and expensive. It should be a compiler, not a runtime. |

A method that only documents its successes is marketing. These five are the parts most
likely to matter to you, because they are the parts that were missing.

**The method has also been reviewed against itself, and lost.** An independent review found
that the grammar could not express the repo's own examples, that the flagship idempotency
example passed at t=0 against exactly the defect it was written for, that store reads were
simultaneously forbidden-as-evidence and used as primary evidence, and that the hardest
tenancy example had a constructible broken system that passed it. All four are fixed, and
each produced a new catalogue entry —
[A4](method/12-false-greens.md), [B5](method/12-false-greens.md),
[B6](method/12-false-greens.md), [C5](method/12-false-greens.md). Two of them are false
greens **created by this method's own rules**, which is the category worth hunting hardest.

## How to adopt this

```
1. Read method/01 through method/04.               (~20 minutes)
2. Copy templates/ into your repo as specs/.
3. Fill in templates/environment.yaml for one environment.
4. Give your agent prompts/author-specs.md and one endpoint.
5. Review the spec it produced against adoption/checklist.md.
6. Give your agent prompts/execute-specs.md and run it.
7. Read the evidence bundle. Not the verdict — the bundle.
```

Step 7 is the one people skip and the one that matters. The first few times, verify the
verifier.

Full walkthrough: [`adoption/getting-started.md`](adoption/getting-started.md).

## What this is not

- **Not a replacement for unit tests.** Different economics, different failure modes.
  Keep them for logic.
- **Not a load, concurrency, or race detector.** A scenario runner will not find a race.
  Pair with a load tool and accept the boundary.
- **Not a fuzzer.** SpecOps verifies enumerated claims. For input-space coverage, pair
  with a property-based tool driven off your API schema.
- **Not a UI testing method.** It is API- and infrastructure-first. Browser interaction is
  one more evidence source, not the subject.
- **Not vendor-specific.** Nothing in `method/` names a cloud, a model, or a log
  aggregator. Those live in `examples/` and your own environment bindings.

## Status

**v0 — the method is stable enough to use and not yet stable enough to freeze.**
Page numbers in `method/` are intended to be citable; they will not be renumbered without
a deprecation note. Everything else may change.

Contributions, and especially disagreements backed by a concrete failure, are welcome —
see [`CONTRIBUTING.md`](CONTRIBUTING.md).

## License

MIT.
