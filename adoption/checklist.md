# Adoption checklist

Two checklists. The first is for a single specification, at review time. The second is for
the suite as a whole, quarterly.

---

## Per specification

### Can it fail?

- [ ] For each scenario, you can describe a broken system that the scenario would catch.
- [ ] Every negative or isolation scenario **seeds a real subject** before asserting it is
      unreachable — and asserts the subject survived.
- [ ] Every absence assertion has a **positive control** over the identical predicate.
- [ ] No scenario's only assertion is a status code the endpoint returns for many reasons.

### Altitude

- [ ] No class, function, or handler names.
- [ ] No file paths or line numbers.
- [ ] No table names, index names, or key shapes.
- [ ] No internal type or DTO names; request and response bodies use **real wire keys**
      with real casing.
- [ ] It would survive a full internal refactor with no observable change.

### Provenance

- [ ] Every assertion is tagged `@intent` or `@contract`.
- [ ] Every `@intent` tag carries a **resolvable citation** — a requirement, ticket, or
      design-document reference.
- [ ] For every status code, limit, timeout, and threshold, you can say where the number
      came from.
- [ ] No assertion exists because it "seemed right."
- [ ] No `@intent` assertion's expected value was edited to match observed behavior while
      keeping its tag. Re-cite, or downgrade to `@contract`.

### Assertions

- [ ] Exact literals wherever the value is deterministic.
- [ ] Stable machine-readable codes, not message copy.
- [ ] Every field a client would depend on is asserted.
- [ ] No code or status was invented; each was verified.

### Side effects

- [ ] Every known downstream effect has an `expect side-effect` block, separate from the
      response block.
- [ ] Every side-effect assertion pins a **per-execution-unique** correlator.
- [ ] No correlator is a time window.
- [ ] **Presence** assertions: no predicate is broad enough to match a previous execution's
      leftovers.
- [ ] **Absence** assertions: the predicate is **broad** over a unique subject — a narrow
      field pin creates misses, not safety.
- [ ] Counters are asserted as "moved", never an exact delta.
- [ ] Every asynchronous assertion names a budget class from the environment file, never a
      literal duration.

### Temporal semantics

- [ ] Every assertion shaped **"still equals" / "exactly N" / "unchanged" uses `hold`**, not
      `within`. With `within` it is satisfied at t=0, before the system could misbehave.
- [ ] Every `hold` is paired with **evidence the stimulus was received** — otherwise
      zero-delivery satisfies the invariant trivially.
- [ ] Every survival or unchanged assertion reads a strongly consistent path, waits out the
      staleness bound, or uses a **product read** rather than a store read.

### Negative and inverted scenarios

- [ ] Every seeded-subject negative scenario has a **seed-visibility control**: the
      legitimate owner reads the seed through the product and gets `200`, before anyone
      asserts it is unreachable.
- [ ] Every `absent through` assertion has a `control` block.
- [ ] The control is of the **correct kind** — same-subject (identical predicate, drained,
      count-of-survivors) or proxy (different predicate, not drained, residual gap stated).
- [ ] The control is **bracketed** — re-asserted after the absence poll — or the residual
      window is stated.
- [ ] A failed control is `FAIL` reason `control`, never `INFRA`.

### Isolation and lifecycle

- [ ] Every precondition is declared in `requires`.
- [ ] Every mutating scenario has a `teardown`, and the teardown removes reverse mappings.
- [ ] Destructive scenarios mint their own subjects; they never consume shared fixtures.
- [ ] Everything created is uniquely named per execution.
- [ ] Ordered dependencies on siblings are marked as chains.

### Structure

- [ ] The header table is complete.
- [ ] Nothing outside a typed block decides a verdict.
- [ ] Notes are test-design rationale, not changelog or run stamps.
- [ ] Referenced scenario ids exist.

---

## Per suite — review quarterly

### Verdict health

- [ ] All seven verdict counts are reported, every run.
- [ ] `KNOWN-DEVIATION` is never folded into `PASS`, and every one carries a resolvable
      citation for the intent it contradicts.
- [ ] Every exactly-once obligation is asserted with a **receipt and a bracket**, not with
      `exists` or a bare `count == n` ([`method/12`](../method/12-false-greens.md) H1).
- [ ] Boundary probes are derived from **declared types** — the narrowest declaration in the
      path — not chosen by judgment ([`method/12`](../method/12-false-greens.md) H2).
- [ ] No percentage is published without its breakdown.
- [ ] `BLOCKED` is computed from `requires`, never judged.
- [ ] Every `BLOCKED` and `N/A` was re-adjudicated against its declared reason this
      quarter.
- [ ] The `BLOCKED` count is not trending upward.
- [ ] `INFRA` is tracked separately and is not being absorbed into "flakiness."

### Retry discipline

- [ ] Re-runs are counted and reported.
- [ ] No retry of a `FAIL` has been normalized.
- [ ] Every asynchronous assertion has a budget; none was extended to obtain a pass.
- [ ] Steps running close to their budgets are surfaced as findings.

### Provenance

- [ ] The `@intent` / `@contract` ratio is computed and published.
- [ ] The behaviors you would least like to be wrong carry `@intent` assertions.
- [ ] New features written spec-first are `@intent` by construction.

### Evidence

- [ ] Bundles are archived, redacted at capture, and gitignored.
- [ ] A verdict can be recomputed from its bundle without touching the system.
- [ ] Every assertion appears in its bundle, including the ones that passed.
- [ ] Runs that gated a promotion are pinned beyond the normal retention window.

### Gates

- [ ] A static gate runs pre-merge and is blocking.
- [ ] The index is generated, not hand-maintained.
- [ ] Header-table fields are checked against deployed configuration.
- [ ] No specification references a production identifier directly.
- [ ] No coverage percentage is used as a blocking threshold.

### Environment

- [ ] Specifications contain no hostnames, regions, physical store names, or cluster
      commands. Logical names only.
- [ ] Every environment declares a safety class, and the runner enforces it.
- [ ] Escape-hatch **writes** are impossible outside a `mutable` environment. (Store
      **reads** are evidence, not escape hatches.)
- [ ] A routing preflight runs at the start of every run.
- [ ] The configuration the corpus assumes is declared and asserted at run start.
- [ ] Configuration is diffed between environments before promoting, and divergent keys are
      named in the run ledger's "not run" section.

### Executor privilege

- [ ] The executor has a **dedicated identity** per environment — not a human's, not the
      application's.
- [ ] Its grants are scoped to exactly the stores, topics, and streams the bindings name.
- [ ] `read-only` environments hand it read-only credentials, so a runner bug cannot become a
      production mutation.
- [ ] Its access is audited under a distinguishable identity.
- [ ] Credentials are short-lived and rotated.
- [ ] Someone **decided** to grant an autonomous process this access. It was not a side
      effect of adopting a test method.

### The evaluator

- [ ] The evaluator has been run against the conformance cases, or you have audited your
      agent's verdicts against them by hand.
- [ ] `within` and `hold` produce different verdicts on the same bundle.
- [ ] A bundle missing an assertion yields `FAIL`, not `PASS`.
- [ ] An empty probe result yields `FAIL`; only a recorded probe error yields `INFRA`.

### The honest questions

- [ ] **When did this suite last catch a real defect?** If the answer is "never", it is
      either verifying nothing or your system is unusually good. Find out which.
- [ ] **When did it last fail for a non-defect reason?** If often, it is being weakened —
      find where.
- [ ] **What did the last incident do that this suite did not catch?** Add the entry to
      your local false-green section.
- [ ] **Which scenarios have never once failed?** Some are fine. Some cannot fail. Check
      the ones you would most rely on.
