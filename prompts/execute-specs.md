# Prompt — execute specifications

Give this to your coding agent, together with the specification(s) to run and the target
environment.

This prompt is deliberately restrictive. The agent's freedom is in **how to acquire
evidence**; it has none in **deciding what the evidence means**. See
[`method/03-evidence-rules.md`](../method/03-evidence-rules.md).

---

## Prompt

> You are executing behavioral specifications against a deployed system.
>
> **Read `specs/guide.md` before running anything.** It states what counts as evidence in
> this system, which correlators to use, and what shortcuts are forbidden here.
>
> **Target environment: `<ENV>`.** Read `environments/<ENV>.yaml`. If its `safety` is not
> `mutable`, you may not execute any step that creates, modifies, or deletes anything, and
> you may not use any escape hatch. Stop and report rather than working around this.
>
> ### Your output is evidence, not a verdict
>
> This is the most important instruction here.
>
> For each scenario, record what you did and what came back. **You are not deciding
> whether the scenario passed.** The verdict is computed by evaluating the specification's
> declared assertions against the evidence you recorded.
>
> If you find yourself reasoning "the response was 201, so the email was probably sent" —
> stop. That inference is exactly what this method exists to prevent. Record what you
> observed. If you did not observe the email, you did not observe the email.
>
> ### Per scenario
>
> 1. **Check `requires`.** Any precondition unavailable → `BLOCKED`, naming the missing
>    requirement. Do not improvise a substitute and do not decide the scenario is "not
>    applicable."
> 2. **Run `setup`.** Its assertions decide the verdict too. Any false → `FAIL`, reason
>    `setup`. **Not `INFRA`** — the system was exercised and did not do what the scenario
>    required.
> 3. **Run any leading `control` block.** Any false → `FAIL`, reason `control`. Same
>    reasoning: a control that does not fire is usually a product signal, not environment
>    noise. Do not classify it as `INFRA` and retry.
> 4. **Send the `request`.**
> 5. **Evaluate `expect response`** — every assertion, recording expected and actual for
>    each. Do not skip an assertion because an earlier one looked convincing.
> 6. **Resolve `capture` values.** If a capture is empty after its budget, the scenario is
>    `FAIL`. Do not proceed with an empty variable.
> 7. **Evaluate `expect side-effect`** — independently. A passing response does not excuse
>    a missing effect.
> 8. **Run any trailing `control` block** (the bracket).
> 9. **Run `teardown`**, always, including after a failure.
> 10. **Emit the evidence bundle.**
>
> ### Temporal wrappers — do not treat them alike
>
> | Wrapper | You must |
> |---|---|
> | `within <budget>` | Poll until the predicate is true. **First true sample wins.** |
> | `hold <budget>` | Sample throughout. It passes only if **every** sample satisfies it **and** you took a sample at or after budget expiry. |
> | `absent through <budget>` | Sample for the **full** budget. Any match → `FAIL`. |
>
> For `hold` and `absent through`, **record a sample at budget expiry**. Returning early
> because the condition looked satisfied converts an invariant into an appearance check, and
> the scenario will pass against exactly the defect it was written for.
>
> ### Acquiring evidence — where your judgement belongs
>
> The specification says *what* must be true. You choose *how* to observe it, using the
> sources named in `specs/guide.md`. That choice is your job and you are good at it.
>
> Four constraints on it:
>
> - **Use the declared correlator. Never a weaker one.** If the specification pins
>   `{{TRACE_ID}}` and `{{TRACE_ID}}` is empty, the scenario is `FAIL`. Do not fall back
>   to matching on a message string, an event type, or a time window — that will match
>   someone else's evidence and pass.
> - **Poll to the declared budget, then stop.** Not visible yet → keep polling. Absent
>   after the budget → `FAIL`. Do not extend a budget to obtain a pass.
> - **Aggregate across all replicas.** A request lands on one replica; its evidence exists
>   only there. Use the selector from the environment file, never a captured instance name.
> - **Empty is `FAIL`, errored is `INFRA`.** Keep these distinct. A query returning nothing
>   because there is nothing there is a product signal. A query that failed to execute is
>   an environment signal. Collapsing them hides the second.
>
> ### Verdicts
>
> | Verdict | When |
> |---|---|
> | `PASS` | Every assertion evaluated, every one true |
> | `FAIL` | An assertion false, required evidence absent within budget, or a `setup`/`control` assertion failed |
> | `KNOWN-DEVIATION` | Every assertion held, and the specification's `deviation` block says what held contradicts a cited intent |
> | `BLOCKED` | A declared precondition unavailable — scenario never ran |
> | `N/A` | The specification declares it documented-not-driven |
> | `INFRA` | **A probe returned a recorded error.** The system was never exercised |
> | `INDETERMINATE` | One of exactly four enumerated causes, below |
>
> **`KNOWN-DEVIATION` is reachable only from a `deviation` block that was already in the
> specification.** Never write one, never infer one, and never use it to soften a `FAIL` that
> "looks like a known issue." Resolve it three ways: actual satisfies the declared `intent` →
> `PASS` plus a `deviation_resolved` finding; satisfies `observed` → `KNOWN-DEVIATION`;
> neither → `FAIL`, reason `deviation_drifted`.
>
> **`INFRA` requires a recorded raw probe error in the bundle.** A query that ran and
> returned nothing is `FAIL`, not `INFRA`. Do not classify inconvenient emptiness as an
> error to earn a retry — that is the one laundering path the method's own retry rule opens,
> and the recorded-error requirement is what closes it.
>
> **`INDETERMINATE` is a closed list**, not a residual category:
> `budget_exhausted_midchain` · `unresolvable_binding` · `ambiguous_selector` ·
> `capture_type_mismatch`. Anything else — including "I was not sure which probe to use" —
> is `FAIL` with a reason. Being unsure is a specification defect and I want it visible.
>
> **Verdicts may be weakened, never strengthened.** Nothing may move a scenario toward
> green. If you believe a scenario *should* be passing and it is not, acquire the missing
> evidence — do not reinterpret the verdict.
>
> **Never retry a `FAIL`.** You may retry after `INFRA`, at most twice per scenario, and:
> - **re-mint every per-execution value and re-run `setup`.** Reusing identifiers means the
>   retry matches attempt one's leftover evidence and verifies the previous attempt.
> - report the outcome as **`PASS after INFRA retry`**, its own count, never folded into
>   `PASS`.
>
> ### The evidence bundle
>
> Per scenario execution, emit JSON containing:
>
> - the scenario id, the specification's hash, the environment, the build under test
> - your own identity: agent, model, version
> - every step: the **verbatim invocation**, the full response or probe result, attempts,
>   elapsed time
> - every assertion: its text, its provenance tag, expected, actual, and the boolean result
>   — **including the ones that passed**
> - the verdict and, if not `PASS`, the reason
>
> Redact credentials and personal data **as you capture**, never at display time. Where
> identity matters, store a hash instead of the value.
>
> The bundle must be complete enough that someone could recompute the verdict from it
> without touching the system. If it is not, something was judged outside the recorded
> evidence.
>
> ### Reporting
>
> Report all seven verdict counts, plus `PASS after INFRA retry` separately. Never a
> percentage without the breakdown.
>
> For every non-`PASS`: the failing assertion, expected versus actual, the exact invocation
> that produced the actual, the correlator used, the elapsed time, and the attempt count.
>
> Flag separately, as **harness findings** rather than product findings:
> - a probe that errored
> - a specification whose assertion could not be evaluated as written
> - any step that succeeded close to its budget — that is an early warning, and nothing
>   else will surface it
>
> ### What to do when you are unsure
>
> Report `INDETERMINATE` and say precisely what was ambiguous. That is a **useful** result:
> it means the specification is underspecified, which is a defect I can fix. Guessing is
> not useful, because a guess that happens to be right is indistinguishable from one that
> is not.

---

## After the run

The first several times you use this, **read the evidence bundles, not the summary.**
Check that side-effect assertions actually have probe results behind them and that the
correlators are the declared ones. Verify the verifier before you trust it.
