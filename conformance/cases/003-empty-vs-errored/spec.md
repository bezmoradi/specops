# Case 003 — empty is `FAIL`, errored is `INFRA`

**Catches:** [`method/12`](../../../method/12-false-greens.md) C5 — INFRA laundering. This is
one of the two false greens the method's **own** retry rule creates, so an evaluator that
gets it wrong undoes a rule the method depends on.

**The distinction under test.** `FAIL` and `INFRA` are separated by exactly one thing: a
**recorded raw probe error**. A probe that executed and found nothing is a statement about
the product. A probe that could not execute is a statement about the environment. They look
identical in a summary and mean opposite things — and only one of them earns a retry.

The pressure is real and one-directional. `INFRA` is retryable and blocks nothing; `FAIL`
stops the pipeline. An executor under any pressure to be green has a standing incentive to
call emptiness an error, and nothing in the method's prose can stop it. The recorded-error
requirement is the only thing that can, because it is checkable from the bundle alone.

---

## Fragment A — the probe ran and found nothing

```expect side-effect
store(refunds).item({{REFUND_ID}})
  within cross_service
  exists
```

Against step `p-A`: null at every sample through expiry, `error: null`.

Expected: **`FAIL`**, reason `absent_within_budget`.

## Fragment B — the probe could not run

Identical assertion. Against step `p-B`: no samples, and a recorded error —
`connection refused` from the store endpoint.

Expected: **`INFRA`**, `infra_error` carrying the recorded error verbatim.

The system was never exercised, so there is no product signal here in either direction. This
is the *only* verdict in the algebra that says "ask again."

## Fragment C — `INFRA` claimed with nothing behind it

Identical assertion. Against step `p-C`: null at every sample, `error: null` — and the
executor recorded `"verdict_hint": "INFRA"` on the step.

Expected: **`FAIL`**, reason `absent_within_budget`, **plus** harness finding
`infra_without_recorded_error`.

The evaluator does not accept the executor's classification. It recomputes from the evidence,
finds a probe that ran and returned nothing, and returns `FAIL`. A `verdict_hint` is data
about the executor, not an input to the verdict.

**This fragment is the whole case.** An evaluator that trusts a declared `INFRA` has handed
the verdict back to the agent — which is [R1](../../../method/03-evidence-rules.md) violated
at the one point where violating it is invisible, because the scenario simply gets retried
and nobody reads a retry.

---

## What this case does not cover

Retry *accounting* — the two-attempt cap and reporting `PASS after INFRA retry` as its own
count — is executor behavior across runs. The evaluator is pure and sees one execution, so it
cannot check it. Audit that from the run log instead
([`method/02-verdicts.md`](../../../method/02-verdicts.md)).
