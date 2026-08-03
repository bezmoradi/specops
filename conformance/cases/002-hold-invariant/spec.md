# Case 002 — `hold` versus `within` on an invariant

**Catches:** [`method/12`](../../../method/12-false-greens.md) B6 — an invariant asserted
with appearance semantics, satisfied at t=0 before the system could have misbehaved.

**The distinction under test.** The bundle below records a refund count that is `1` at the
first two samples and `2` at the third. An evaluator that treats `hold` like `within` — first
true evaluation wins — returns `PASS`. The correct verdict is `FAIL`.

This is the exact shape of a double-refund defect surviving an idempotency scenario.

---

## Fragment A — the correct form (`hold`)

```expect side-effect
store(refunds).count where order_id == {{ORDER_ID}}
  hold cross_service
  == 1
```

Expected: **`FAIL`** — sample at `t=12000ms` has `count == 2`, and `hold` requires the
predicate at **every** sample.

## Fragment B — the incorrect form (`within`), included for contrast

```expect side-effect
store(refunds).count where order_id == {{ORDER_ID}}
  within cross_service
  == 1
```

Expected: **`PASS`** — `within` requires only that **some** sample satisfies the predicate,
and the samples at `t=0` and `t=1500` do.

**Both expectations are correct behavior for the evaluator.** The bug is not in the
evaluator; it is in a specification that chose `within`. That is why
[`09-gates.md`](../../../method/09-gates.md) gates the choice statically: no assertion
shaped `count ==` / `still` / `unchanged` may use `within`.

An evaluator that returns the same verdict for A and B is non-conformant, whichever verdict
it returns.
