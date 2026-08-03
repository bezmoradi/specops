# Case 008 — an invariant with no sample at expiry is `FAIL`

**Catches:** the loophole left open by [`002`](../002-hold-invariant/spec.md). Case 002 makes
an evaluator distinguish `hold` from `within`. This case closes the obvious way to satisfy
`hold` cheaply: stop sampling early and every sample taken is, trivially, true.

**The distinction under test.** `hold` and `absent through` are claims about a **duration**.
Neither can be settled by evidence that stops before the duration does. So both require a
sample at or after budget expiry, and the absence of that sample is absence of evidence,
which [R2](../../../method/03-evidence-rules.md) makes `FAIL`.

Note where the asymmetry with `within` sits. A `within` **pass** needs no final sample — it
is settled the moment the predicate is true. A `within` **fail** does need one, and so does
every `hold` and `absent through` outcome. The general form: **a sample at expiry is
required whenever the verdict is a claim about the whole budget.**

---

## Fragment A — `hold`, truncated

```expect side-effect
store(refunds).count where order_id == {{ORDER_ID}}
  hold cross_service
  == 1
```

Against step `p-A`: `1` at `t=0`, `t=1500`, `t=4000`. Budget is 30 000 ms. No further
samples.

Expected: **`FAIL`**, reason `missing_final_sample`.

Every recorded sample satisfies the predicate. An evaluator implementing `hold` as
"`all(samples)`" returns `PASS` — and it has just reproduced the exact defect case 002
exists to catch, by a different route. The double refund arrives at `t=12000`, in the window
nobody looked at.

## Fragment B — `absent through`, truncated

```expect side-effect
event(billing).published where {{MARKER}} appears anywhere
  absent through cross_service
```

Against step `p-B`: no match at `t=0`, `t=1500`, `t=4000`. Nothing further.

Expected: **`FAIL`**, reason `missing_final_sample`.

Same rule, and worth stating separately because `absent through` is where truncation is
most tempting: nothing is happening, so there is nothing to wait for. That intuition is
backwards. An absence assertion is *entirely* a claim about the window, so a shortened window
is a proportionally weaker claim, and this is the assertion form whose failure mode is
permanent green.

## Fragment C — `hold`, sampled to expiry

Against step `p-C`: `1` at `t=0`, `1000`, `2000`, … through `t=30000`.

Expected: **`PASS`**. The baseline, so that an evaluator cannot pass this case by failing
every `hold`.

## Fragment D — sampled to expiry, but twice

Against step `p-D`: `1` at `t=0` and `1` at `t=30000`. Nothing between. The environment
declares `poll_interval: 1s`.

Expected: **`PASS`**, **plus** harness finding `sampling_interval_not_honored`.

This is the residual gap, and it is here to be named rather than fixed by fiat. The
assertion is satisfied under the stated rule — every sample is true and a sample exists at
expiry — so `PASS` is the correct verdict and an evaluator must not invent a density
threshold to reject it. But a `hold` with two samples is a `within` with extra steps: a
transient violation at `t=12000` passes through it untouched.

What makes this checkable rather than a matter of taste is that the interval is **declared**
in the environment binding ([`10-environments.md`](../../../method/10-environments.md)), not
chosen by the evaluator. An evaluator that has the binding compares against it and reports the
shortfall. One that does not have the binding reports nothing — and should say so, rather
than guess.
