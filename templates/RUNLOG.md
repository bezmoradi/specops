# Run log — <system name>

Append-only record of verification runs, newest first.

**Why this exists.** Evidence bundles answer *what* happened; this answers *what we
decided and what we did not run.* It is the only artifact that answers "was this green
before the incident?" in a form a human can read six months later, and no amount of
tooling substitutes for someone writing down what they chose not to do.

**Keep entries to a summary.** Per-scenario contracts live in the specifications;
per-execution evidence lives in the bundles. This file holds counts, notable
confirmations, and — most importantly — **what was not run and why.**

---

## <YYYY-MM-DD> (run #<n>) — <one-line purpose>

**Environment.** <name> · routing preflight <passed/failed> · <anything unusual about the
environment this run>

**Build under test.** `<commit or image>` — <what changed since the last run, and why this
run was worth doing>

**Result.** <n> PASS · <n> FAIL · <n> BLOCKED · <n> N/A · <n> INFRA · <n> INDETERMINATE
across <n> lanes, with <n> re-runs.

<!--
Report ALL SIX counts. A "97% pass rate" is compatible with a silently growing BLOCKED
pile. Re-runs are reported explicitly because a retry after INFRA is legitimate and a
retry after FAIL is the anti-pattern — the number keeps that distinction visible.
See method/02-verdicts.md.
-->

**Provenance.** <n>% of executed scenarios were `@intent`.

<!--
State what kind of confidence this run provides. "563 passed" is compatible with a suite
that has never verified a single requirement. See method/05-provenance.md.
-->

**Per-lane.** LANE-01 <p>/<f>/<b> · LANE-02 <p>/<f>/<b> · …

**Adjudication of non-PASS verdicts.**

<!--
Every BLOCKED and N/A re-checked against its specification's own declared reason. These
are the two verdicts that decay into fiction if nobody re-reads them; when one no longer
matches its declared reason, the specification changed under you.
-->

| Verdict | Scenario | Declared reason | Still accurate? |
|---|---|---|---|
| BLOCKED | | | |
| N/A | | | |

**Notable confirmations.**

<!-- Behaviors this run specifically proved, especially ones a recent change put at risk. -->

**Findings — product.**

<!-- Real defects. Link to wherever they were filed. -->

**Findings — harness.**

<!--
Drift, rot, and defects in the suite itself. Distinct from product findings and just as
important: a broken probe produces INFRA at best and a false green at worst.
See method/12-false-greens.md section F.
-->

**Performance.**

<!--
Wall clock, replay-vs-plan fraction, and any step running close to its budget. A step
succeeding on its seventh attempt at 9.8s of a 15s budget is one deploy from red, and
nothing else in the run will tell you. See method/11-economics.md.
-->

**Teardown.** Steady state verified: <yes/no> — <what remained>

**Not run.** <!-- Everything skipped, and why. The most important line in the entry. -->

---
