# Case 005 — an inverted verdict is only as good as its control

**Catches:** [`method/12`](../../../method/12-false-greens.md) G1 — an intended non-event with
no positive control. This is the false green with the longest half-life in the catalogue: it
passes forever, including after the publishing pipeline has been dead for months, and nothing
in the run report will ever look wrong.

**The distinction under test.** Absence is equally well explained by "the feature works" and
"the observation path is dead." A control is what separates them. Three things follow, and
an evaluator must handle each:

1. An inverted assertion with **no** control is a **corpus finding**, not a verdict change.
   The evaluator computes the verdict from evidence as always — it does not get to
   retroactively fail a scenario for being badly written. Reporting it is what makes the
   gate ([`09-gates.md`](../../../method/09-gates.md)) able to reject it statically.
2. A **failing** control is `FAIL`, reason `control` — **never** `INFRA`. This is the fork
   where the entire mechanism is won or lost, and §"Why not INFRA" below is the argument.
3. A control that passes at the front and fails at the back is still `FAIL`. The bracket is
   load-bearing; a front-only control proves the pipeline was live when the window *opened*.

---

## Fragment A — absence holds, no control declared

```expect side-effect
event(billing).published where {{MARKER}} appears anywhere
  absent through cross_service
```

Against step `p-A`: sampled to expiry, no match. The specification declares no `control`
block.

Expected: **`PASS`**, **plus** corpus finding `inverted_assertion_without_control`.

The `PASS` is correct and the finding is mandatory. Both. An evaluator that fails the
scenario has started judging specifications instead of evidence; an evaluator that stays
silent has certified the single most durable false green in the catalogue as clean.

## Fragment B — the control does not fire

```control
event(billing).published where {{CONTROL_MARKER}} appears anywhere
  within cross_service
  count == 1
```

Against step `p-B-ctl`: sampled to expiry, no match, `error: null`. The absence poll was
never run.

Expected: **`FAIL`**, reason `control`. The inverted assertion is reported
`evaluated: false`.

### Why not `INFRA`

Because the control is a *product action*. In the worked example the control is tenant B
cancelling its **own** order, which the product is supposed to publish. It did not. The
system was exercised and did not do what the scenario required — that is a product signal
wearing environment clothing.

The consequence of getting this wrong is not cosmetic. `INFRA` is retryable and blocks
nothing, so a genuine publishing regression gets two automatic retries and then a line in a
harness-findings section nobody triages. `FAIL` stops the pipeline and someone looks. If it
turns out to be the harness, that adjudication happens in the open, afterwards — which is the
correct order, because the alternative pre-classifies every control failure as noise before
anyone has looked at it.

## Fragment C — control fires, absence holds, bracket confirms

Leading control matches at `t=1200`; absence poll clean through expiry; trailing control
matches at `t=31500`.

Expected: **`PASS`**, no findings. This is the only shape in the case that is genuinely
green, and it is here so that an evaluator cannot pass the other fragments by being
uniformly pessimistic.

## Fragment D — the pipeline died during the window

Leading control matches at `t=1200`; absence poll clean through expiry; **trailing control
does not match**.

Expected: **`FAIL`**, reason `control`.

The absence assertion is `true` and it is worthless: the observation path was alive when the
window opened and dead when it closed, so the clean absence poll covers an unknown fraction
of the window. A scenario that returns `PASS` here is reporting "the event did not fire" when
the honest statement is "nothing could have been seen to fire."
