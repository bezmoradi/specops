# Case 009 — exactly-once needs a receipt *and* a bracket

**Catches:** [`method/12`](../../../method/12-false-greens.md) H1 — an exactly-once
obligation asserted by a form that "at least one" satisfies. This case was written because a
real corpus, applying every other rule in the method correctly, shipped a double-publish
defect.

**The distinction under test.** Three forms look like they assert "exactly one event." Only
one of them does. An evaluator must return **different** verdicts for A, B and C against the
same bundle — an evaluator that agrees with itself across them has collapsed the distinction
the case exists to draw.

The bundle records a stimulus that publishes **twice**: once at `t=120`, once at `t=4100` —
a delayed duplicate, the shape a retry or a redelivery produces. Both carry the same
per-execution correlator.

---

## Fragment A — existence

```expect side-effect
event(billing).published where order_id == {{ORDER_ID}}
  within cross_service
  exists
```

Expected: **`PASS`**.

Correct evaluator behavior, and a defective specification. `exists` is true of one message
and equally true of two. It cannot fail on a duplicate, so it is not an exactly-once
assertion at all — it is a liveness check wearing one's clothes.

## Fragment B — cardinality with appearance semantics

```expect side-effect
event(billing).published where order_id == {{ORDER_ID}}
  within cross_service
  count == 1
```

Expected: **`PASS`**.

**This is the fragment that matters.** `count == 1` looks like the fix, and the receipt
carve-out correctly assigns it `within` — the subject did not exist before the stimulus, so
this is an appearance claim, not an invariant, and B6 has nothing to say about it.

But `within` is first-true-wins. The count reaches 1 at `t=120`, the assertion is satisfied,
and polling stops. The duplicate at `t=4100` arrives after the evaluator stopped looking.

So `count == 1 within` catches only a duplicate already in flight before the first sample —
the *simultaneous* case. It is blind to every delayed duplicate, which is where duplicates
actually come from. An author who writes this has done the obvious repair and is still
unprotected.

## Fragment C — the pair

```expect side-effect
event(billing).published where order_id == {{ORDER_ID}}
  within cross_service
  count == 1

event(billing).published where order_id == {{ORDER_ID}}
  hold single_hop
  count == 1
```

Expected: **`FAIL`**, reason `assertion_false`, on the second assertion, violating sample at
`t=4100`.

The receipt bounds the count upward from nothing; the bracket bounds it forward in time.
Neither alone is an exactly-once claim.

Note the bracket's budget is `single_hop`, not `cross_service`: it should cover the system's
**retry or redelivery interval**, which is the window a duplicate has to arrive in. If that
interval is unknown, it is worth discovering — not guessing.

---

## The corpus-level half

Fragments A and B are correct evaluator behavior on defective specifications. Nothing an
evaluator does can rescue them, because there is no false assertion to catch — the assertion
that would have failed was never written.

That is why H1 is also a **static gate check** and a **completeness obligation**
([`09-gates.md`](../../../method/09-gates.md)):

- The gate flags a positive assertion on a stimulus-produced, uniquely-correlated subject,
  outside a `control` block, with no bracketing wrapper in the scenario — emitting
  `exactly_once_unbracketed`. It fires on A and B, not on C.
- The completeness check generates the obligation from the **event catalogue**, so an event
  declared exactly-once with no scenario asserting it that way is reported even when the
  corpus contains no relevant assertion to inspect.

An evaluator alone cannot close this class. That is the lesson H1 records.
