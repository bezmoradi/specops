# 11 — Economics

> If a model must reason through every scenario on every run, the suite is slow, expensive,
> and non-reproducible. None of those are inherent to the method.

The agent's value is in *figuring out how* to verify something. That is a one-time cost per
scenario, not a per-run cost — and treating it as per-run is the main reason
agent-executed suites get abandoned.

## The agent is a compiler, not a runtime

```
first execution     agent explores: which probe answers this? what is the correlator?
                    how long does it take? what does the evidence look like?
                    └─► on success, the resolved plan is CACHED

later executions    the cached plan is replayed deterministically
                    no model in the loop
                    └─► agent re-invoked only on: spec change, plan error, or assertion failure
```

A **plan** is the resolved, concrete version of a scenario: exact requests, exact probe
invocations, exact correlator extraction, exact budgets. It is what the agent worked out,
frozen.

**Is a frozen plan just a brittle scripted test again?** The objection is fair and the
answer is a distinction, not a denial. A cached plan *is* a script — but one whose script
was derived under this method's constraints and is **re-derivable on demand**. Traditional
brittleness comes from a hand-written script nobody can regenerate, so when it breaks the
only options are patching it or deleting it. Here the plan is a build artifact of a
specification that still exists, and the invalidation triggers below are what keep
"re-derivable in principle" from becoming "nobody has re-derived it since March."

**One honest dependency:** replay presupposes a runner that can execute a cached plan. This
method deliberately ships no implementation, so this page describes a capability you must
either build or obtain. The conformance kit
([`09-gates.md`](09-gates.md)) is what makes building one tractable — it specifies the
behavior a runner must exhibit and gives it test cases.

What this buys:

- **Reproducibility.** A replayed plan does the same thing every time. Non-determinism is
  confined to the first execution and to re-planning.
- **Speed and cost.** Replay runs at the speed of the network. The dominant cost disappears.
- **Gate eligibility.** A cached suite is fast enough to gate more than a promotion.
- **Durability.** If models get slower, more expensive, or unavailable, cached plans keep
  working. The method survives its own dependency.

## Invalidation

A plan must be discarded when it may no longer be right:

| Trigger | Why |
|---|---|
| The specification's hash changed | The assertions or the stimulus changed |
| The environment binding changed | Hosts, stores, budgets, or selectors moved |
| **The routing preflight failed** | Surfaces have moved; every cached invocation may target the wrong service ([`10-environments.md`](10-environments.md)) |
| **Assumed configuration diverged** | A flag the corpus depends on changed; plans encode behavior under the old value |
| A probe invocation errored | The evidence path itself changed |
| An assertion failed | Either a real regression, or the plan is stale — both need the agent |
| Age exceeds a threshold | Plans silently rot as systems evolve; expire them deliberately |

**A failed assertion must never auto-invalidate into a silent re-plan that then passes.**
That is re-run-until-green wearing a different hat. On failure, report the failure. Re-plan
as a **separate, explicit** action whose result is visible: "this scenario failed, and
re-planning made it pass" is important information about your specification, not a green
run.

## Tiering a run

Not every scenario deserves the same treatment.

| Tier | What | When |
|---|---|---|
| **Static** | No system, no model — the corpus gate ([`09-gates.md`](09-gates.md)) | Every pull request |
| **Replay** | Cached plans, deterministic, no model | Every deploy |
| **Plan** | Agent explores and caches | New scenarios; invalidated plans |
| **Explore** | Agent runs with latitude, no cache | Investigating a failure; adding coverage |

A healthy steady state is mostly replay. If most runs are still planning, either the
specifications are churning — worth understanding — or plans are being invalidated too
eagerly.

## Where the cost actually goes

When runs are expensive, the cost is rarely the assertions. It is usually:

- **Fixture provisioning.** Often the single largest cost. Parallelize it, and provision
  only the fixtures the selected scenarios declare that they require.
- **Evidence acquisition round-trips.** A separate remote query per scenario is slow.
  **Batch them:** have each scenario *record* its expected evidence, and run one sweep per
  lane at the end that fetches each stream once and checks every recorded correlator. This
  turns N round-trips into one and is usually the biggest single win available.
- **Waiting out budgets.** Budgets should be as tight as the environment honestly allows.
  A 30-second budget where 5 would do costs 25 seconds on every asynchronous assertion.
- **Serial execution.** See [`07-isolation.md`](07-isolation.md).

## When not to use an agent at all

Being honest about this makes the method more credible, not less:

- **A pure request/response assertion with no evidence-acquisition question** — a status
  code, a schema shape, an authentication rejection — needs no agent. A conventional
  declarative API test tool does it faster and cheaper. Use one.
- **Anything running on every commit at high frequency.** Static and replay tiers only.
- **Load, fuzzing, and property-based testing.** Different tools. See the scope boundary in
  [`01-overview.md`](01-overview.md).

The agent earns its cost where the verification question is genuinely open: *where does
the proof of this live, in this environment, right now?* That is a smaller fraction of a
suite than it first appears, and concentrating the agent there is what makes the whole
thing affordable.

## Measure the run

Track these per run, from the bundles:

| Metric | Why |
|---|---|
| Wall clock, total and per lane | Finds the long pole |
| Fraction replayed vs. planned | Cache health |
| Invalidations, by trigger | Which trigger is churning |
| Attempt counts against budgets | Early warning: a step at 90% of budget is one deploy from red |
| `INFRA` rate | Environment health, kept separate from product health |
| Cost per run, and per scenario | The number that decides whether this survives a budget review |

The attempt-count metric is the most underrated. A scenario succeeding on its seventh
attempt at 9.8 seconds of a 15-second budget is not passing comfortably — it is about to
start failing, and nothing else in the run will tell you.
