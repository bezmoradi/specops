# 08 — Evidence bundles

> A method that argues "evidence over assertion" and then archives nothing is not
> auditable. It is asking to be trusted, which is the thing it set out to replace.

This page describes a practice most suites do not follow, including suites that follow
every other page here. It is the difference between a run that *produced* a verdict and a
run that can *defend* one.

## The problem

The usual output of an agent-executed run is a summary: counts, a per-scenario verdict,
and prose describing what happened. That is enough to act on today and useless in six
weeks, when the question is:

- Was this scenario green before the incident, and green against *what evidence*?
- This scenario has failed twice in ten runs. Same cause both times?
- The team changed the log format last month. Which scenarios were silently relying on it?
- This has passed a hundred times. Has it ever actually evaluated its side-effect assertion?

None of these are answerable from a summary. All are answerable from an archive.

There is a sharper version of the last one. Under R1 the agent does not vote — but
something still has to confirm the agent *acquired what it claimed to acquire*. Without a
recorded bundle, "the assertion was evaluated" is itself an unverified claim.

## What a bundle contains

One bundle per scenario execution. Machine-readable, diffable, complete enough to
re-evaluate the verdict without re-running anything.

```json
{
  "scenario": "orders-create#OR3",
  "spec_hash": "sha256:…",
  "run_id": "2026-08-01T09:14:22Z-a63dc1b",
  "environment": "staging",
  "build_under_test": "a63dc1b",
  "executor": { "agent": "…", "model": "…", "version": "…" },
  "started_at": "…", "finished_at": "…",

  "steps": [
    {
      "kind": "request",
      "probe": "http",
      "invocation": "POST https://…/v1/orders",
      "request":  { "headers": {…}, "body": {…} },
      "response": { "status": 409, "headers": {…}, "body": {…}, "duration_ms": 84 }
    },
    {
      "kind": "capture",
      "variable": "ORDER_ID",
      "resolved": "01J…",
      "attempts": 3,
      "elapsed_ms": 2140
    },
    {
      "kind": "probe",
      "probe": "log",
      "invocation": "<recorded, verbatim>",
      "correlator": { "trace_id": "4bf9…" },
      "attempts": 7,
      "elapsed_ms": 9800,
      "result_count": 1,
      "excerpt": "…"
    }
  ],

  "assertions": [
    { "source": "expect response",    "text": "status == 409",
      "provenance": "@intent",   "expected": 409, "actual": 409, "result": true },
    { "source": "expect side-effect", "text": "log(orders).line where trace_id == {{TRACE_ID}} .event == \"order.rejected\"",
      "provenance": "@contract", "expected": "order.rejected", "actual": "order.rejected", "result": true }
  ],

  "verdict": "PASS",
  "verdict_reason": null
}
```

## The guarantees it must provide

**1. The verdict is recomputable.** Given the bundle and the specification, the verdict
must be derivable without touching the system. If it is not, something judged outside the
recorded evidence — which is an R1 violation, and the bundle just made it visible.

**2. Every assertion appears, including the ones that passed.** An assertion that is
absent from the bundle was not evaluated. Silent skipping is the failure mode this catches,
and there is no other way to catch it.

**3. Probe invocations are recorded verbatim.** Not "queried the logs" — the actual
invocation. This is what makes a failure reproducible by a human and what exposes an agent
that quietly widened a predicate.

**4. Attempts and elapsed time are recorded per polled step.** A step that succeeded on the
seventh attempt at 9.8 seconds against a 15-second budget is one deploy away from being a
red. That signal exists nowhere else, and it is the earliest warning you get.

**5. The executor is identified.** Agent, model, version. When verdict behavior changes and
no specification did, this is the first thing to check.

**6. The specification is hashed.** A bundle from a specification that has since changed
must not be silently compared against a current one.

## Redaction

Bundles contain live traffic. They will contain credentials, tokens, and personal data
unless something prevents it.

- **Redact at capture, never at read.** A bundle written with a token in it has already
  leaked; deciding to hide it at display time does not help.
- **Redact by shape and by name.** Authorization headers, cookies, anything matching a
  credential pattern. Default to redacting unknown headers on sensitive routes rather than
  allow-listing after the fact.
- **Store a hash of a redacted value where identity matters.** You often need to know that
  two steps used *the same* token without knowing what it was.
- **Bundles are gitignored.** They belong in run storage, not the repository.

## Retention

- **Recent runs in full.** Enough to compare against, typically a few weeks.
- **A permanent index.** Scenario, run, build, verdict, duration, attempt counts. Small,
  cheap, and enough to answer "when did this start getting slower?" indefinitely.
- **Pin the bundles for any run that gated a promotion.** If a deploy proceeded because a
  run was green, the evidence for that decision should outlive the normal window.

## What this unlocks

Once bundles exist, several things that were manual become mechanical:

**Run diffing.** Two runs of the same specification against different builds produce
comparable bundles. The diff shows exactly which assertion's actual value moved — far more
useful than two verdicts.

**Flake detection.** A scenario whose attempt counts trend upward over successive runs is
degrading. You see it before it fails.

**Coverage from traces.** If bundles record the trace identifiers a scenario produced, and
your tracing knows which code served those traces, you can compute which scenarios
exercised which parts of the system — from real execution rather than a hand-maintained
map. That is the only version of change-driven selection that does not silently rot, and it
falls out of bundles for free ([`09-gates.md`](09-gates.md)).

**Explainability.** A failure report becomes: the assertion, the expected and actual
values, the exact invocation that produced the actual, the correlator it used, how long it
waited, and how many times it tried. That is a report an engineer can act on without
re-running anything — which is the difference between a suite people use and a suite people
route around.
