# Case 010 — `KNOWN-DEVIATION` resolves three ways

**Catches:** a known defect reported as `PASS`, and its mirror — a *fixed* defect reported as
`FAIL`, which punishes the repair.

**The distinction under test.** A `deviation` block declares both the cited `intent` and the
`observed` behavior. The evaluator compares the **actual** against both and resolves three
ways. An evaluator that implements fewer than three has reintroduced the problem the verdict
exists to solve.

This case comes from a real collision between two of the method's own rules:
[`05-provenance.md`](../../../method/05-provenance.md) says record what the system does and
tag it `@contract`; [`12`](../../../method/12-false-greens.md) G2 says a corpus of
`@contract` assertions is a regression net, not an oracle. With a known defect between them,
asserting the intended value gives permanent red and asserting the observed value gives a
green suite on a system known to be broken.

---

## The specification under test

```deviation
assertion: status
intent:    == 422    cite: api-contract#validation-errors
observed:  == 500
```

```expect response
status == 500
body.error == "Database operation failure"
```

The scenario asserts `observed`, so it does not rot. The block records what the contract
says.

---

## Fragment A — the deviation persists

Against step `s-A`: actual status `500`.

Expected: **`KNOWN-DEVIATION`**, carrying the cited intent, counted separately.

**Not `PASS`.** Every assertion held, so a naive evaluator returns `PASS` and the exit code
reports a system with a known defect as fine — the exact laundering this verdict exists to
prevent. The number must appear in the run report as its own count.

**Not `FAIL`.** The scenario is doing what it was written to do. Forcing red here is the
alternative the method rejects: permanent red trains a team to ignore red, and the trained
behavior generalizes to the reds that matter.

## Fragment B — the deviation is fixed

Against step `s-B`: actual status `422`, `body.error == "Required field"`.

Expected: **`PASS`**, plus corpus finding `deviation_resolved`.

**This fragment is why the verdict is three-way rather than two-way.** Under a plain
`@contract` assertion of `status == 500`, fixing the product turns this scenario **red** —
the suite punishes the repair, and whoever fixed the bug inherits a broken pipeline. Here the
evaluator sees the actual satisfying the declared `intent`, returns green, and tells the
author to delete the now-obsolete block and retag the assertion `@intent`.

Note the scenario's own `expect response` assertions are false in this fragment
(`status == 500` against an actual of `422`). The deviation resolution **overrides** them —
this is the one place in the method where a false assertion does not produce `FAIL`, and it
is safe precisely because the alternative outcome is the one that discourages fixing bugs.

## Fragment C — the behavior moved somewhere else

Against step `s-C`: actual status `503`.

Expected: **`FAIL`**, reason `deviation_drifted`.

Neither the cited intent nor the declared observation. Something changed that nobody
declared, and the declaration is now stale in a way that is not a fix. This is the fragment
that stops `deviation` from becoming a blanket exemption for a route.

## Fragment D — a deviation with no citation

Same as A, but the block's `intent` carries no `cite:`.

Expected: **`KNOWN-DEVIATION`**, plus corpus finding `deviation_without_citation`.

The verdict is unchanged — the evaluator judges evidence, not specification quality — but
without a citation the block is one engineer's opinion that the system is wrong, which is not
a deviation from anything. Same treatment as an `@intent` tag lacking a citation.

---

## The constraint that makes this safe

`KNOWN-DEVIATION` is the only route by which something that would otherwise be `FAIL` becomes
softer, so it is the one an executor under pressure will reach for.

**The block must exist in the specification the run was planned from.** An evaluator must not
accept a `deviation` block injected into a bundle and must never synthesize one from a
failure that "looks like a known issue." A conformant evaluator given fragment C's bundle plus
a `deviation` block claiming `observed: == 503` — written after the fact — must reject it as a
harness finding rather than downgrade the `FAIL`.
