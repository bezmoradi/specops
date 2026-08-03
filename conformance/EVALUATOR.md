# Evaluator conformance

Under [R1](../method/03-evidence-rules.md), the evaluator decides every verdict. An
evaluator that is subtly wrong is therefore a **silent, total** failure — the suite reports
verdicts that mean something other than what everyone believes.

This directory specifies evaluator behavior and provides golden cases so any evaluator — a
program, or an agent following [`../prompts/execute-specs.md`](../prompts/execute-specs.md)
— can be tested against it.

**This is a specification, not an implementation.** It is the answer to "the method says the
agent must not vote, but ships nothing that stops it." It does not stop it. It makes an
adopter able to **audit** their executor in an afternoon, and gives any future runner a
definition of correct.

## Inputs and output

```
evaluate(specification, evidence_bundle) → verdict + per-assertion results
```

The evaluator is **pure**: it touches no system and performs no acquisition. Everything it
needs is in the bundle. If a verdict cannot be derived from the bundle alone, something was
judged outside recorded evidence — an R1 violation, which the conformance suite is designed
to expose.

## Normative behavior

### Ordering

1. Resolve `requires`. Any unmet → `BLOCKED`, reason = the requirement name. **Stop.**
2. If the specification declares `N/A` → `N/A`. **Stop.**
3. Evaluate `setup` assertions. Any false → `FAIL`, reason `setup`. **Stop.**
4. Evaluate leading `control` assertions. Any false → `FAIL`, reason `control`. **Stop.**
5. Resolve `capture`. Any unresolved that a later assertion depends on → `FAIL`, reason
   `empty_capture`. **Stop.**
6. Evaluate `expect response` — **all** of them, recording each.
7. Evaluate `expect side-effect` — **all** of them, recording each.
8. Evaluate trailing `control` (bracket) assertions.
9. Resolve any `deviation` block (below).
10. `PASS` iff every evaluated assertion is true and no deviation persists. Otherwise `FAIL`,
    or `KNOWN-DEVIATION` per step 9.

### Deviation resolution

A `deviation` block declares an `assertion`, an `intent` expression with a citation, and an
`observed` expression. The scenario's own assertion tracks `observed`, so the evaluator has
the actual value in hand and resolves three ways:

| Actual satisfies | Verdict | Also emits |
|---|---|---|
| `intent` | `PASS` | corpus finding `deviation_resolved` |
| `observed` | `KNOWN-DEVIATION` | counted separately; never folded into `PASS` |
| neither | `FAIL`, reason `deviation_drifted` | — |

Two constraints, both mechanical:

- **The block must be present in the specification the run was planned from.** An evaluator
  must not accept a `deviation` block injected into the bundle, and must not synthesize one.
  This is the only route by which a `FAIL` could become something softer, so it is the one
  that gets checked.
- **`intent` without a resolvable citation is a corpus finding**
  (`deviation_without_citation`), reported and not verdict-changing — same treatment as an
  `@intent` tag lacking one.

`teardown` results are recorded and reported but do not change the verdict.

**Steps 6–8 never short-circuit.** An assertion absent from the results was not evaluated,
and an evaluator that stops at the first failure cannot be distinguished from one that
skipped.

### Temporal semantics

| Wrapper | Passes iff |
|---|---|
| `within <budget>` | **Any** sample in the bundle satisfies the predicate |
| `hold <budget>` | **Every** sample satisfies it, **and** a sample exists at or after budget expiry |
| `absent through <budget>` | **No** sample satisfies it, **and** a sample exists at or after budget expiry |

`hold` and `absent through` **fail** if the bundle contains no sample at or after expiry —
absence of the final sample is absence of evidence, and R2 makes that `FAIL`. Case
[`008`](cases/008-hold-missing-final-sample/spec.md) is this rule; without it, `hold` is
satisfied by sampling for one second and stopping.

The general form, which also covers `within`:

> **A sample at or after budget expiry is required whenever the verdict is a claim about the
> whole budget.**

So a `within` **pass** needs no final sample — it is settled the moment the predicate is
true — but a `within` **fail** asserts the predicate was false for the entire budget, and
that does. A truncated failing poll stays `FAIL` (a verdict may be weakened, never
strengthened) and is reported as `budget_not_honored`; see case
[`001`](cases/001-within-appearance/spec.md) fragment C.

Where the environment binding declares a `poll_interval`, an evaluator that has the binding
compares the sample count against it and reports `sampling_interval_not_honored`. It does
**not** invent a density threshold of its own, and an evaluator without the binding reports
nothing and says so.

### FAIL versus INFRA

`INFRA` requires a **recorded raw probe error** on the step. A step that ran and returned an
empty result is `FAIL`. An evaluator must not infer `INFRA` from emptiness, and an `INFRA`
without a recorded error is reported as a **harness finding**, not accepted.

### INDETERMINATE

Only these four, each a recorded, structured cause:

`budget_exhausted_midchain` · `unresolvable_binding` · `ambiguous_selector` ·
`capture_type_mismatch`

Anything else is `FAIL` with a reason.

### Provenance

Provenance tags do not affect the verdict. An `@intent` tag lacking a citation is reported
as a **corpus finding** ([`09-gates.md`](../method/09-gates.md)), not a verdict change.

## Reasons and findings

Three output channels, and keeping them separate is what stops the evaluator from quietly
becoming a judge of specification quality.

**Verdict reasons** — a closed vocabulary. Every non-`PASS` verdict carries exactly one.

| Reason | Verdict | Meaning |
|---|---|---|
| `assertion_false` | `FAIL` | An evaluated assertion was false |
| `absent_within_budget` | `FAIL` | Required evidence never appeared, polled to expiry |
| `missing_final_sample` | `FAIL` | A whole-budget claim with no sample at expiry |
| `empty_capture` | `FAIL` | A capture a later assertion depends on was unresolved |
| `assertion_not_evaluated` | `FAIL` | A declared assertion has no result in the bundle |
| `setup` | `FAIL` | A `setup` assertion was false |
| `control` | `FAIL` | A `control` assertion was false, leading or trailing |
| *the requirement name* | `BLOCKED` | That declared requirement was unavailable |
| `deviation_drifted` | `FAIL` | The actual matched neither the declared `intent` nor the declared `observed` |
| `probe_error` | `INFRA` | A recorded raw probe error; the system was never exercised |
| one of the four causes | `INDETERMINATE` | `budget_exhausted_midchain` · `unresolvable_binding` · `ambiguous_selector` · `capture_type_mismatch` |

`KNOWN-DEVIATION` carries the cited intent it contradicts, not a reason code.

`N/A` carries the specification's declared reason.

**Harness findings** — the run was defective, independently of the verdict. Reported
alongside it; they never change it.

| Code | Raised when |
|---|---|
| `infra_without_recorded_error` | `INFRA` claimed with `error: null` |
| `weaker_correlator` | The issued selector omits the declared correlator |
| `budget_not_honored` | A poll stopped short of expiry |
| `sampling_interval_not_honored` | Samples are sparser than the binding's `poll_interval` |
| `evaluation_short_circuited` | Evaluation stopped at the first false assertion |
| `blocked_without_unmet_requirement` | `BLOCKED` claimed with every requirement available |

**Corpus findings** — the *specification* is defective. Also never a verdict change; these
are what a static gate ([`09-gates.md`](../method/09-gates.md)) should have caught earlier.

| Code | Raised when |
|---|---|
| `inverted_assertion_without_control` | An `absent through` assertion with no `control` block |
| `intent_without_citation` | An `@intent` tag with no requirement reference |
| `deviation_resolved` | A `deviation` block whose `intent` is now satisfied — delete it and retag |
| `deviation_without_citation` | A `deviation` block whose `intent` has no resolvable citation |
| `exactly_once_unbracketed` | A positive assertion on a stimulus-produced, uniquely-correlated subject, outside a `control` block, with no bracketing `hold` / `absent through` in the scenario — [`12`](../method/12-false-greens.md) H1 |

An executor's `verdict_hint` is **data about the executor**, never an input. The evaluator
recomputes and, where the two disagree, reports the disagreement as a harness finding. This
is the single point where R1 is easiest to lose, because a verdict handed back to the agent
here is invisible: the scenario is simply retried, and nobody reads a retry.

## Golden cases

Each directory contains `spec.md`, `bundle.json`, and `expected.json`. Run your evaluator
over the pair; disagreement is a bug in the evaluator, not a matter of opinion.

All eight are written. Most carry several **fragments** sharing one bundle — near-identical
inputs that must produce **different** outputs, because "returns the same verdict for both"
is a far more common bug than "returns the wrong verdict for one."

| Case | Asserts |
|---|---|
| `001-within-appearance` | `within` passes on **any** satisfying sample; a failing `within` still needs a sample at expiry |
| `002-hold-invariant` | The FC2 class: a count already true at t=0 **fails** under `hold` when a later sample violates it, and **passes** under `within` — the evaluator must not treat them alike |
| `003-empty-vs-errored` | Empty result → `FAIL`; recorded probe error → `INFRA`; a declared `INFRA` with no recorded error → `FAIL` plus a finding |
| `004-empty-capture` | Unresolved capture → `FAIL`, never a broader-predicate retry — **including when every recorded assertion says `true`** |
| `005-absent-through-no-control` | No `control` block → corpus finding, verdict unchanged; a failing control → `FAIL` reason `control`, **not** `INFRA`; a failing *trailing* control → `FAIL` even though the absence assertion is true |
| `006-blocked-computed` | Unmet `requires` → `BLOCKED` even if everything else passed; met `requires` → `BLOCKED` is unreachable, however the executor labelled it |
| `007-no-short-circuit` | A declared assertion missing from the bundle → `FAIL`, not `PASS`, even though every *present* assertion is true |
| `008-hold-missing-final-sample` | `hold` and `absent through` with no sample at expiry → `FAIL`, even when every recorded sample satisfies the predicate |
| `009-exactly-once` | `exists` and `count == 1 within` both **pass** against a delayed duplicate; only receipt-plus-bracket fails. The first two emit `exactly_once_unbracketed` |
| `010-known-deviation` | Three-way deviation resolution: persists → `KNOWN-DEVIATION`; fixed → `PASS` + `deviation_resolved`; neither → `FAIL` reason `deviation_drifted` |

Cases `002`, `005`, and `007` are the three that most implementations get wrong. Start
there — then run `008` fragments A and C, which are the cheapest way to find out whether
`hold` was implemented as `all(samples)`.

## Contributing a case

A case is `(spec fragment, bundle, expected verdict)` plus one sentence naming the mistake it
catches. Cases that encode a false green from
[`12-false-greens.md`](../method/12-false-greens.md) are the most valuable — that is how a
catalogue entry becomes mechanically checkable rather than advisory.
