# Case 007 — a missing assertion is not a passing assertion

**Catches:** [`method/12`](../../../method/12-false-greens.md) A2 — the agent narrates
instead of evaluating. Of everything in the kit this is the one an evaluator is most likely
to get wrong **by accident**, because "iterate the results and check they are all true" is
the obvious implementation and it is wrong.

**The distinction under test.** The evaluator's input is a specification **and** a bundle. A
result set that is all-true answers "did anything recorded fail?" — which is a different
question from "was every declared assertion evaluated?" The gap between those two questions
is the entire false green: an assertion that never ran cannot fail, and an evaluator that
folds absence into truth turns skipping work into a way to be green.

Every assertion in the specification must appear in the results, **including the ones that
passed**. That requirement in [`08-evidence-bundles.md`](../../../method/08-evidence-bundles.md)
reads like bookkeeping. It is not. It is the only thing that makes this case decidable.

---

## The specification under test

```expect response
status == 201
body.code == "ORDER_CREATED"
body.data.id exists
body.data.status == "pending"
body.data.total_cents == 2000
```

Five assertions. Hold that number.

---

## Fragment A — four results, all true

The bundle records results for `status`, `body.code`, `body.data.id`, and
`body.data.status`. There is no result for `body.data.total_cents`.

Expected: **`FAIL`**, reason `assertion_not_evaluated`, naming
`body.data.total_cents == 2000`.

Nothing in this bundle is false. Every number matches, the response is a healthy `201`, and
a summary generated from it reads as a clean pass. The response body in the bundle even
contains `total_cents: 500` — so the assertion, had it been evaluated, would have failed.
That detail is deliberately present and the evaluator must **not** use it: the verdict is
`FAIL` because the assertion is *absent*, not because a re-derivation says it would have
been false. An evaluator that starts re-deriving skipped assertions from raw evidence is
doing acquisition, and on a side-effect assertion it could not do it at all.

## Fragment B — five results, one false

All five appear. The third is false. The fourth and fifth are present and true.

Expected: **`FAIL`**, reason `assertion_false`, with **five** assertion results in the
output.

This is a conformant bundle of a failing scenario, and it is here as the contrast: the
verdict matches fragment A's, and the two are not the same event. One is a product defect
with a complete record; the other is an incomplete record.

## Fragment C — the evaluator stopped at the first failure

All five assertions are declared. The bundle records three: the first two true, the third
false. Nothing after it.

Expected: **`FAIL`**, reason `assertion_false`, **plus** harness finding
`evaluation_short_circuited` naming the two that never ran.

The verdict is right and the run is still defective. Two assertions were not evaluated, so
this scenario's report cannot distinguish "one thing is broken" from "at least one thing is
broken and we stopped counting." When the third assertion is fixed, the fourth and fifth
become the next surprise — a defect that was present the whole time and arrives looking like
a regression.

This fragment is why [`EVALUATOR.md`](../../EVALUATOR.md) says steps 6–8 never short-circuit.
The cost of the rule is a few wasted probes on a scenario already doomed. The benefit is that
one failing run tells you everything that is wrong instead of the first thing.

---

## The one-line test for an evaluator

> Does it read the specification at all, or only the bundle?

An evaluator that only reads the bundle **cannot** pass fragments A or C, whatever else it
does correctly — it has no way to know five assertions were declared. That is the structural
version of this case, and it is worth checking before running anything.
