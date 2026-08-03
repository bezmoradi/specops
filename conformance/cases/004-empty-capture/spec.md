# Case 004 — an unresolved capture is `FAIL`, not a broader search

**Catches:** [`method/12`](../../../method/12-false-greens.md) B1 — fallback to a weaker
correlator. This is the highest-frequency false green in agent-executed suites, because the
fallback is *helpful*: the agent wanted to find the evidence and did.

**The distinction under test.** [R3](../../../method/03-evidence-rules.md) says an
unresolved capture is itself a `FAIL`. It is not a prompt to search harder. A correlator
exists to make evidence provably **this execution's**; a broadened predicate finds evidence
that belongs to someone else, and it does so most reliably in a busy environment, which is
where you least want it.

The evaluator can catch this from the bundle alone, and that is worth stating plainly: it
sees the declared capture is empty *and* sees the invocation that was actually issued.

---

## The specification under test

```capture
TRACE_ID = response.header["X-Trace-Id"]
```

```expect side-effect
log(orders).line where trace_id == {{TRACE_ID}}
  within single_hop
  count >= 1
```

---

## Fragment A — capture empty, executor stopped

Against step `s-A`: the response carried no `X-Trace-Id` header, so `TRACE_ID` resolved to
`""`. The executor did not issue the side-effect probe.

Expected: **`FAIL`**, reason `empty_capture`, naming `TRACE_ID`. The side-effect assertion is
reported as `evaluated: false`.

Note that this **stop is not a short-circuit**. Ordering step 5 in
[`EVALUATOR.md`](../../EVALUATOR.md) halts evaluation on an unresolved capture by design, and
the halt is recorded. That is different from case [`007`](../007-no-short-circuit/spec.md),
where assertions silently went missing inside a block that must never stop early.

## Fragment B — capture empty, executor broadened the predicate

Against step `s-B` and probe `p-B`: `TRACE_ID` again resolved to `""`, and the executor
issued

```
log(orders).line where message contains "order created"
```

which matched three lines. The recorded assertion says `count >= 1`, actual `3`,
`result: true`.

Expected: **`FAIL`**, reason `empty_capture`, **plus** harness finding `weaker_correlator`.

The recorded `result: true` is not honored. The declared correlator was `{{TRACE_ID}}`; the
issued invocation does not contain it; the capture it depends on is empty. Three log lines
matched — from other executions, from an earlier run, possibly from a different tenant. The
assertion was true of *something*, which is precisely the failure mode.

**A conformant evaluator returns `FAIL` here even though every recorded assertion is `true`.**
That sentence is the case.

## Fragment C — capture resolved, but to the wrong shape

```capture
ITEM_COUNT = response.body.data.items
```

```expect side-effect
store(orders).item({{ORDER_ID}})
  within index_propagation
  .item_count == {{ITEM_COUNT}}
```

Against step `s-C`: `ITEM_COUNT` resolved to an **array** of two objects, and the persisted
`item_count` is the number `2`. A field equality whose right-hand side is a captured value
requires the two to be comparable.

Expected: **`INDETERMINATE`**, cause `capture_type_mismatch`.

Not `FAIL`: the assertion is not false, it is unevaluatable, and the difference matters
because the repair is to the *specification*, not to the product. This is the one fragment in
the kit that exercises an enumerated `INDETERMINATE` cause — evidence that the verdict is
computed from a closed list rather than used as a shrug.
