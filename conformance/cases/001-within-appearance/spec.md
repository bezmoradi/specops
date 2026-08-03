# Case 001 — `within` is an appearance check

**Catches:** the complement of [`002`](../002-hold-invariant/spec.md). An evaluator that
gets `hold` right by making *both* wrappers strict is equally non-conformant — it turns
every legitimate eventual-consistency assertion red and the suite gets muted.

**The distinction under test.** `within` is satisfied by **any** sample. It does not require
a sample at budget expiry, and it must not require every sample to satisfy the predicate.
But that asymmetry cuts one way only: a `within` that **fails** is a claim that the predicate
was false for the *whole* budget, and that claim needs a sample at or after expiry behind it.

---

## Fragment A — appearance arrives late

```expect side-effect
event(billing).published where order_id == {{ORDER_ID}}
  within cross_service
  .event_type == "order.created"
```

Against step `p-A`: null at `t=0`, null at `t=1500`, present at `t=9330`.

Expected: **`PASS`**, `satisfying_sample_at_ms: 9330`.

An evaluator that requires all samples to satisfy returns `FAIL` here. That is the
over-correction this case exists to catch.

## Fragment B — never appears

```expect side-effect
event(billing).published where order_id == {{ORDER_ID}}
  within cross_service
  .event_type == "order.created"
```

Against step `p-B`: null at every sample through `t=30000`, `error: null`.

Expected: **`FAIL`**, reason `absent_within_budget`.

Note what this is **not**. The probe ran and returned nothing — that is a product signal, so
it is `FAIL`, never `INFRA` (see [`003`](../003-empty-vs-errored/spec.md)). And the budget is
not extended to obtain a different answer.

## Fragment C — the same failure, on a truncated poll

Identical assertion. Against step `p-C`: null at `t=0`, `t=1500`, `t=4000`, and nothing
further, against a 30 000 ms budget.

Expected: **`FAIL`**, reason `absent_within_budget`, **plus** harness finding
`budget_not_honored`.

The verdict is unchanged, and that is the point: truncating a poll can only produce a false
**red**, never a false green, so [`02-verdicts.md`](../../../method/02-verdicts.md)'s
weakened-never-strengthened rule leaves `FAIL` standing. An evaluator must not "repair" this
into `INDETERMINATE` — being short of evidence for a red is not one of the four enumerated
causes. It must, however, say so: an unreported truncation is how a suite quietly starts
polling for four seconds and nobody learns why the flake rate moved.
