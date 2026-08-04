# 02 — Verdicts

A run reports one of seven verdicts per scenario. Most suites have two. The extra ones exist
because collapsing them destroys information you need specifically when something is wrong.

| Verdict | Meaning | Decided by |
|---|---|---|
| `PASS` | Every declared assertion evaluated true against recorded evidence | Computed |
| `FAIL` | An assertion evaluated false, or required evidence was absent within budget | Computed |
| `KNOWN-DEVIATION` | Every assertion held, and what held **contradicts a declared intent** | Computed from a `deviation` block |
| `BLOCKED` | A declared precondition was unavailable. The scenario never ran. | Computed from `requires` |
| `N/A` | The specification declares this scenario documented-not-driven | Declared in the spec |
| `INFRA` | A probe or the environment errored. The system was never exercised. | Computed from a probe error |
| `INDETERMINATE` | A declared, enumerable acquisition failure occurred | Computed from a closed list |

**Every verdict is computed.** None is a judgement call. Where the boundary between two
verdicts is subtle — `FAIL` versus `INFRA`, `FAIL` versus `INDETERMINATE` — the boundary is
defined mechanically below, because every judgement-shaped boundary becomes a drain that
failures escape through.

## PASS

**Every assertion in the scenario was evaluated, and every evaluation returned true.**
There is no other path to green.

Scoped precisely: this is a claim about the **judgement step**. No reasoning, retry, or
adjudication may move a scenario toward green. It is *not* a claim that the pipeline cannot
be fooled — see the threat model in [`01-overview.md`](01-overview.md#threat-model).

Not `PASS`: assertions skipped because an earlier step looked convincing; a side effect
that "must have happened" because the response was `2xx`; a scenario narrated as successful
without its `expect` blocks being evaluated.

If you find yourself writing "effectively passed," the verdict is `FAIL` or
`INDETERMINATE`.

## FAIL

Three causes, all `FAIL`:

- **Contradiction.** Evidence was acquired and an assertion evaluated false.
- **Absence.** Evidence an assertion depends on was not present within its budget.
- **Setup or control failure.** A `setup` or `control` block's assertion did not hold, so
  the stimulus was never validly delivered or the observation path was never proven live.

The third is deliberate and worth dwelling on. A failed positive control is **not** `INFRA`
— the system *was* exercised and something in it did not do what the scenario required.
Classifying it as environment noise takes a product regression and files it where nothing
blocks on it. Report `FAIL` with reason `control`, and adjudicate harness-versus-product in
the open.

Absence being `FAIL` is the fail-closed rule
([`03-evidence-rules.md`](03-evidence-rules.md) R2). The instinct — "we could not find the
log line, so we cannot say" — produces a suite that goes green whenever observability
breaks, which is exactly when you need it.

A `FAIL` report carries the failing assertion, expected, actual, and the evidence it was
evaluated against. A `FAIL` with only a message will be dismissed.

## KNOWN-DEVIATION

**Every assertion held, and the specification declares that what held contradicts a
documented intent.**

This verdict exists because of a collision between two of this method's own rules, found the
hard way. [`05-provenance.md`](05-provenance.md) says to record what the system does today
and tag it `@contract`. [`12-false-greens.md`](12-false-greens.md) G2 says a corpus of
`@contract` assertions is a regression net, not an oracle. Put a *known defect* between them
and the rules point opposite ways:

- Assert the **intended** behavior and the scenario is permanently red. Permanent red trains
  a team to ignore red, which is a worse outcome than the defect.
- Assert the **observed** behavior and the suite reports green on a system you know is
  broken. The exit code is the only channel a pipeline reads, and it now says "fine."

Neither is acceptable, and no amount of prose in the specification fixes it, because prose
does not reach the exit code. The missing thing was never a rule — it was a **verdict**.

### The `deviation` block

```
​```deviation
assertion: status
intent:    == 422    cite: <published contract reference>
observed:  == 500
​```
```

The scenario asserts the **observed** value, so it stays green-adjacent and does not rot.
The block records what the contract *says*, with a resolvable citation, and the evaluator
resolves three ways — mechanically, from evidence:

| Actual satisfies | Verdict | Also emits |
|---|---|---|
| `intent` | `PASS` | corpus finding `deviation_resolved` — delete the block, retag `@intent` |
| `observed` | **`KNOWN-DEVIATION`** | counted separately, never folded into `PASS` |
| neither | `FAIL`, reason `deviation_drifted` | the behavior moved to a third thing nobody declared |

The first row is the property that makes this work. Under a plain `@contract` assertion,
**fixing the bug turns the scenario red** — so the suite punishes the repair, and the author
who fixes the product gets a failing pipeline for their trouble. Here, fixing it produces
`PASS` plus an instruction to remove the now-obsolete declaration.

### Rules

- **Never folded into `PASS`.** Reported as its own count, like `PASS after INFRA retry`.
- **Exit policy belongs to the consumer, not the method.** A pre-merge gate may treat
  `KNOWN-DEVIATION` as passing; a release gate may not. What the method fixes is that the
  number is *visible* and cannot be reached by accident.
- **`intent` requires a resolvable citation** — same bar as an `@intent` tag
  ([`05-provenance.md`](05-provenance.md)). A deviation with no citation is one engineer's
  opinion that the system is wrong, and it is a corpus finding.
- **A rising `KNOWN-DEVIATION` count is a backlog, and that is the point.** It converts "we
  know about that one" from tribal memory into a number that appears in every run.

This is the one verdict that is *about the specification's relationship to a requirement*
rather than about the run. It is deliberately narrow: it applies only where the intended
behavior is documented somewhere citable. An undocumented disagreement is not a deviation,
it is an opinion, and it belongs in a bug report.

## BLOCKED

A declared precondition was unavailable: a fixture that does not exist, a credential that
was not provisioned, a sandbox that is down, hardware that is not present.

**Computed from the specification's `requires` block**, checked before execution. The
reason is the name of the missing requirement.

```
​```requires
fixture: second_tenant_account
credential: payment_provider_test_key
​```
```

The failure mode this prevents: an agent that cannot complete a scenario decides it was
"not applicable," and a genuine regression is filed as an environmental limitation. If
`BLOCKED` is a judgement call, it is the drain failures escape through.

A `BLOCKED` count trending upward is a suite decaying. Track it.

### Preconditions may be about regime, not only presence

A `requires` entry is usually a thing that exists or does not: a fixture, a credential, a
sandbox. It may also be a **quantitative regime** — a threshold outside which the
scenario's subject is provably unobservable, whatever the product does. Timing-sensitive
contracts have these: past some latency, some queue depth, some clock skew, a control that
is present and a control that has been deleted produce identical observable behavior, and
no assertion at any sample size separates them
([`12-false-greens.md`](12-false-greens.md) G9).

```
​```requires
regime: observed_elapsed < 85s   # above this, control-present and control-absent are
                                 # observationally identical — derivation in the spec
​```
```

This is not a softening of the rule above; it is the same rule applied to a measured
precondition, and the distinction from a judgement call is auditable. A derived threshold
has arithmetic behind it and is written **before** the run. A judgement call appears for
the first time in the report. If you cannot show the derivation, you have
[`12-false-greens.md`](12-false-greens.md) C2, not a precondition.

Inside such a region `BLOCKED` is the only honest verdict: `PASS` claims a verification
that did not occur, and `FAIL` blames the product for the environment.

## N/A

The specification declares that this scenario documents a behavior a black-box run cannot
drive deterministically — a real third-party delivery, a process signal, a fault-injection
path, a cross-node broadcast.

**Declared in the specification; never inferred at runtime.** A scenario with an `expect`
block is a `PASS`/`FAIL` candidate, always. A scenario without one must say so, with a
reason.

An `N/A` scenario is documentation of a known blind spot in the place someone will look.
Deleting it makes the blind spot invisible; converting it to `PASS` makes it a lie.

## INFRA

**A probe or the environment returned an error.** The system under test was never
exercised.

### Classification is mechanical

This verdict is a sanctioned retry channel, so it must not be an agent's opinion. `INFRA`
requires a **recorded probe error**:

| `INFRA` | Not `INFRA` |
|---|---|
| Non-zero exit from a probe invocation | A query that ran and returned nothing → `FAIL` |
| Transport error, DNS failure, connection refused | A `4xx`/`5xx` **from the system under test** → evaluate the assertion |
| Credential rejected by the evidence source | Credential rejected by the system under test → evaluate the assertion |
| Timeout of the probe mechanism itself | Budget expiry with the probe working → `FAIL` |
| A binding that does not resolve | A control assertion that failed → `FAIL`, reason `control` |

**The bundle must record the raw probe error for every `INFRA`.** An `INFRA` without one is
itself a harness finding — that is the check that stops empty-but-inconvenient from being
reclassified as errored.

### Retries

- Legitimate, and **counted**.
- **Capped** — two per scenario per run; beyond that the verdict stands.
- A retry **re-mints every per-execution value and re-runs setup**. Reusing correlators
  across attempts matches attempt one's residue ([`12-false-greens.md`](12-false-greens.md)
  B5).
- **`PASS after INFRA retry` is reported as its own count**, never folded into `PASS`. A
  suite where that number is climbing is being laundered, and the number is the only thing
  that shows it.

A rising `INFRA` rate is a signal about your environment that will otherwise be absorbed as
"flakiness."

## INDETERMINATE

**A declared, enumerable acquisition failure.** Not a residual category, and not "the agent
was unsure" — that is `FAIL`.

The closed list:

| Cause | Meaning |
|---|---|
| `budget_exhausted_midchain` | A multi-hop chain ran out of budget with hops remaining, so the final assertion was never reached |
| `unresolvable_binding` | A specification referenced a logical name absent from the environment file |
| `ambiguous_selector` | A probe selection matched multiple distinct subjects where the assertion presumes one |
| `capture_type_mismatch` | A captured value could not be coerced to the type an assertion requires |

**Anything not on this list is `FAIL` with a reason.** If an agent cannot decide which
probe answers a question, that is a `FAIL` and an underspecified specification — both of
which you want visible.

`INDETERMINATE` may be treated as `FAIL` for gating. Keeping it distinct tells you the
**specification** is defective rather than the system. Every `INDETERMINATE` is
human-adjudicated per run, alongside `BLOCKED` and `N/A`; a scenario that goes
`INDETERMINATE` twice is a specification defect and the fix is in the spec.

## Reporting rules

1. **Report all seven counts**, plus `PASS after INFRA retry` separately.
2. **Never a percentage without the breakdown.** "97% pass rate" is compatible with a
   silently growing `BLOCKED` pile — or with every known defect parked in
   `KNOWN-DEVIATION`.
3. **Every non-`PASS` names its cause** — the failing assertion, the missing requirement,
   the raw probe error, the enumerated `INDETERMINATE` cause, the declared `N/A` reason, the
   cited intent a `KNOWN-DEVIATION` contradicts.
4. **Adjudicate `BLOCKED`, `N/A`, `INDETERMINATE`, and `KNOWN-DEVIATION` every run.** These
   are the verdicts that decay into fiction if nobody re-reads them. Each should still match
   its specification's declared reason; when it does not, the specification changed under
   you. `KNOWN-DEVIATION` decays the fastest, because a deviation everyone has stopped
   noticing is indistinguishable from an accepted contract.

## The one-way rule

```
PASS             ←  nothing may be promoted to this
KNOWN-DEVIATION  ←  reachable only from a declared `deviation` block, never by inference
FAIL             ←  INDETERMINATE may be treated as this for gating
```

No process, retry, adjudication, or agent reasoning may move a scenario *toward* green. An
agent that "resolves" an `INDETERMINATE` into a `PASS` by reasoning about what probably
happened has defeated the method. If a scenario should be green and is not, acquire the
missing evidence — do not reinterpret the verdict.

`KNOWN-DEVIATION` sits below `PASS` and is subject to the same rule from below: an executor
may **not** downgrade a `FAIL` into `KNOWN-DEVIATION` because the failure "looks like a known
issue." The block must have been in the specification *before* the run, declaring both
values, with a citation. Written after the fact to quiet a red, it is the drain this whole
page exists to prevent — and it is the one this verdict most invites, so the gate checks it
([`09-gates.md`](09-gates.md)).
