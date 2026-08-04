# 12 — False greens

> A suite that reports red when the system is fine is annoying.
> A suite that reports **green** when the system is broken is worse than no suite, because
> it is consuming budget to manufacture confidence.

This is the catalogue. Every entry has been observed in a real agent-executed suite. Every
one produces a *passing* run against a broken or unverified system.

Re-read this page after any incident a green suite failed to catch, and add what you found.

**Format:** symptom → mechanism → rule → what the rule costs you.

---

## A. Inference instead of observation

### A1 — The response implies the side effect

**Symptom.** The scenario passes. The downstream effect never happened.

**Mechanism.** The agent observes `201 Created`, cannot readily find the downstream
evidence, and reasons that since the request succeeded, the effect "probably" occurred.
Each individual inference is plausible. In aggregate, the side-effect half of the suite
verifies nothing.

**Rule.** Response and side effect are independent verdicts. Both must pass. Absent
evidence is `FAIL` ([`03-evidence-rules.md`](03-evidence-rules.md) R2).

**Cost.** Ordinary propagation delay now produces reds. Pay that with poll budgets, not by
relaxing the rule.

### A2 — The agent narrates instead of evaluating

**Symptom.** The report describes a successful run in convincing prose. Some assertions
were never evaluated.

**Mechanism.** Nothing forces every declared assertion to be checked, so an agent that has
formed a view of the outcome writes the report from that view.

**Rule.** The agent never votes. Verdicts are computed from recorded evidence, and every
assertion appears in the bundle — including the ones that passed
([`03-evidence-rules.md`](03-evidence-rules.md) R1, [`08-evidence-bundles.md`](08-evidence-bundles.md)).

**Cost.** Assertions must be written in a machine-evaluatable form.

### A3 — Seeded state used as evidence

**Symptom.** An isolation or authorization scenario passes. The isolation is not enforced.

**Mechanism.** The scenario asserts that a foreign resource is *not* reachable — but never
created one. The request would return "not found" whether isolation works or not, so the
assertion cannot fail.

**Rule.** A negative assertion needs a real subject. Seed the foreign resource, then assert
it is unreachable, then assert it still exists afterward. **The seed is load-bearing** —
label it so nobody optimizes it away.

**Cost.** Escape-hatch setup, and teardown for it.

### A4 — The seed the product cannot see

**Symptom.** An isolation scenario passes. Isolation is not enforced.

**Mechanism.** The scenario seeds a foreign subject by writing directly to the store, then
asserts a caller cannot reach it. But the raw write did not populate everything the
application's read path needs — a tenant-scoped secondary index, a derived field, a cache.
The product returns "not found" for *everyone*, including the legitimate owner. The
negative assertion passes, the survival check passes because the row is physically there,
and the protection under test was never exercised.

This is A3 one level deeper: A3 says do not assert on state you seeded; A4 says the seed
may not exist as far as the product is concerned.

**Rule.** Any scenario that seeds a subject and asserts a caller cannot reach it MUST first
prove the **product** can reach it, using the legitimate owner's credential, in a `setup`
block. See [`06-side-effects.md`](06-side-effects.md#escape-hatches).

**Cost.** The owner's credential must exist and must be usable for a positive read — so a
"this identity is only ever a target, never a caller" convention has to be relaxed.

---

## B. Correlation and timing failures

### B1 — Fallback to a weaker correlator

**Symptom.** Passes consistently. Proves nothing about the request that was made.

**Mechanism.** The declared correlator comes back empty. Rather than failing, the agent
matches on something broader — a message string, an event type, a time window. In a shared
environment that matches *someone else's* evidence: a sibling scenario, ambient traffic, a
previous run.

**Rule.** An empty capture is itself a `FAIL`. The agent may never substitute a weaker
correlator than the specification declares ([`03-evidence-rules.md`](03-evidence-rules.md) R3).

**Cost.** Correlators must be designed rather than improvised.

### B2 — A broad predicate matches a lingering message

**Symptom.** An event assertion passes even when the scenario's action did not publish.

**Mechanism.** The predicate matches only on event type. The queue still holds a matching
message from a previous execution, within its retention window. The oldest match is
returned, indefinitely.

**Rule.** Pin a field unique to this execution — the new resource id, the specific subject,
the minted correlator ([`06-side-effects.md`](06-side-effects.md)).

**Cost.** Every event-producing scenario needs a per-execution-unique field to pin on.

### B3 — Time-window correlation

**Symptom.** Passes under light load, fails under parallelism, passes again on retry.

**Mechanism.** "Evidence produced in the last N seconds" is not a correlator. It collides
the moment anything else runs concurrently — and collisions produce matches, which produce
green.

**Rule.** Never correlate by time. Use a minted id, a propagated trace, or a per-run unique
subject ([`06-side-effects.md`](06-side-effects.md)).

**Cost.** Some evidence paths need work before they can carry an identifier. That work is
the point.

### B4 — Exact counter deltas in a shared environment

**Symptom.** A metric assertion passes, attributing someone else's activity to this
scenario.

**Mechanism.** Metric attributes are deliberately low-cardinality, so a counter cannot
carry a per-execution identifier. Your `+1` shares a bucket with every other occurrence in
the window.

**Rule.** Counters are corroboration only — assert "moved" or "≥ 1", never an exact delta.
The sole exception is a counter keyed by an attribute nothing else emits
([`06-side-effects.md`](06-side-effects.md)).

**Cost.** Counters stop being usable as primary evidence. They were never reliable as such.

### B5 — The retry that reuses correlators

**Symptom.** A scenario fails, is retried after `INFRA`, and passes — against evidence
produced by the first attempt.

**Mechanism.** The retry re-runs the assertions but not the setup, so it reuses the same
minted identifiers, the same subject, the same correlator. Attempt one's evidence is still
sitting in a peeking sink or a log within the query window, and it matches. The retry
verifies attempt one.

This is a false green **created by the method's own retry rule** — `INFRA` sanctions the
retry, and nothing defines whether a retry is a new execution for per-execution-uniqueness
purposes.

**Rule.** A retry re-mints every per-execution value and re-runs `setup`. Where that is
impossible, evidence must be constrained to **postdate the recorded stimulus** of the
current attempt. A freshness bound is a filter that composes with a correlator; it is never
a correlator by itself.

**Cost.** Retries get more expensive, and setup must be idempotent or safely repeatable.

### B6 — An invariant asserted with appearance semantics

**Symptom.** A "must remain exactly N" or "must stay unchanged" assertion passes at the
instant it is made, against a system that violates it a second later.

**Mechanism.** `within` is a poll for *appearance* — the first true evaluation wins. Applied
to an invariant that is **already true when the stimulus is sent**, it is satisfied at t=0,
before the system could possibly have done the wrong thing.

The canonical case is idempotency. Setup waits for the first record to exist, publishes a
duplicate, then asserts `count == 1 within <budget>`. The count is already 1. A system that
double-writes with any processing delay passes every time.

**Rule.** Anything shaped "still equals", "exactly N", or "unchanged" uses **`hold`**, which
requires the condition at every sample and at budget expiry. And `hold` alone does not prove
the stimulus was received — pair it with evidence of receipt (a dedup-hit line, a processed
counter, a consumer acknowledgement), or zero-delivery satisfies the invariant trivially.

**Cost.** The scenario now takes the full budget instead of returning immediately.

### B7 — Stale-read survival checks

**Symptom.** A "the foreign resource still exists / is unchanged" assertion passes against a
system that just destroyed it.

**Mechanism.** The survival read goes through a cache or a read replica and returns
pre-mutation state. Inside the staleness window, a destructive bug is invisible — and
survival checks are exactly the assertions written to catch destructive bugs.

**Rule.** Survival, `hold`, and unchanged assertions read a strongly consistent path, or wait
out the staleness bound before the first sample and say so, or use a **product read** rather
than a store read — the product's consistency guarantee is usually stronger and the altitude
is better.

**Cost.** Slower, and you must actually know your staleness bounds.

---

### B8 — A budget calibrated on the loud direction

**Symptom.** Positive scenarios occasionally fail and pass on re-run; the inverted
scenarios in the same suite are green and have always been green.

**Mechanism.** One knob — "how long to poll for the side effect" — has *opposite* error
modes on either side of the assertion, and only one of them is audible.

- On a **positive** assertion, a too-short budget produces a false FAIL. It is noisy,
  someone re-runs, it passes, and the budget gets blamed and eventually raised.
- On an **inverted** assertion (B6/G1), the identical budget produces a false PASS. The
  unwanted event arrives after the poll gave up; the scenario greens on precisely the
  regression it was written to catch, and reports nothing.

So authors calibrate the number against the failures they can hear, and the inverted
scenarios silently inherit a budget chosen for a different error mode. The budget is also
routinely mis-modelled as "how long the publish takes", when what it must cover is *how
long until the survey has covered the backlog* — a peek-style read that samples a subset of
partitions can miss a message that has been sitting there the whole time, for many rounds,
once the queue is deep.

Observed shape: a 25s budget on a shared async sink produced **5 false FAILs across 4
lanes in a single run**, each costing a re-run to disprove. The same 25s sat unexamined on
the inverted scenarios in that suite, where it had never produced a symptom of any kind.

**Rule.** Derive the budget from the *observation path's* worst case (queue depth ×
sampling behaviour), not from the producer's latency; state it as a floor in the method
rather than per-scenario; and treat any inverted assertion whose budget was inherited from
a positive one as unverified until the floor is applied. A budget that is too short is
never conservative — it is wrong in whichever direction the assertion points.

**Cost.** Slower runs, and a floor someone must justify with a measurement rather than a
guess.

### B9 — The fixture emits the signal under test

**Symptom.** An event scenario is green. It would stay green if the behaviour it names
stopped emitting entirely — because the event it captures is published by the scenario's
own **setup**, not by its stimulus.

**Mechanism.** B2's remedy is "pin a field unique to this execution." That is necessary and
here it is **not sufficient**, which is why this survives a corpus that already applies B2.
The colliding message is not a leftover from a previous run and not a concurrent sibling's
— it is produced by the harness, for the subject the scenario just minted, inside the same
execution, moments before the stimulus. Every per-execution correlator matches it, because
it genuinely belongs to this execution.

Provisioning is the usual source: creating a user emits the account-lifecycle event, making
an org "ready" pushes it through the same status transition the scenario is about to
assert. With first-match-wins correlation the setup's copy is returned and the stimulus is
never observed.

Where it is merely a false *red* (setup and stimulus differ in some field — the reason, the
new status) narrowing the predicate fixes it, and the failure is loud. The dangerous case
is when the fixture's event is **indistinguishable**: same type, same subject, same
discriminator. Then no predicate can separate them, the scenario passes on the setup's
copy, and it is a permanent green over a dead feature. Four scenarios in one file, three
false-red and one false-green, all from the same fixture.

The false-red twin has teeth of its own: it lands on *inverted* scenarios, whose polls are
deliberately broad (G1). There the fixture's event is captured, absence is contradicted,
and a correct system is reported FAIL — so the strongest scenario in the file is the one
most likely to be dismissed as flaky and weakened.

**Rule.** Correlate per **step**, not per execution. Where the fixture can emit the target
event, **drain it before the stimulus** and assert the real capture is a *different*
emission — a distinct `event_id`, not merely a matching predicate. The drain is not
bookkeeping: it doubles as a pre-stimulus positive control, and its absence is what makes
the assertion vacuous. When auditing, ask **"what does my setup publish?"** — the answer is
rarely in the specification, because provisioning is written as plumbing.

**Cost.** Every event scenario needs its provisioning path audited for emissions, and the
drain adds a poll to scenarios that already have one.

### B10 — The setup that prevents the code under test from running

**Symptom.** A scenario reds against a correct product. Or it greens having never executed
the line it exists to cover — and which one you get depends on state unrelated to the
behavior.

**Mechanism.** B9's mirror image. There the fixture *emits* the signal under test; here the
fixture *suppresses the path* to it. A defensive measure in the setup — chosen to make the
scenario safe to run — lands on a branch the product evaluates **before** the code under
test. The stimulus is accepted, the handler returns early on the defensive condition, and
the assertion is evaluated against a system that never reached the subject.

A scenario covering a failure-unwind path (release a dedup sentinel so redelivery can
retry) set its recipient to a blocked test domain "as a second safety net" against sending
real mail. The handler's branch order was claim → blocked-domain skip → preference gate →
send → unwind. The blocked domain returned at branch 2. The unwind at branch 5 never ran,
the sentinel was — correctly — still present, and the scenario failed its primary assertion
against a service whose unwind works perfectly.

**The polarity is luck.** That one reddened, which is loud and gets investigated. Had the
sentinel been absent for any unrelated reason — a TTL, an earlier purge, a colliding key —
the same scenario would have gone green having executed none of the code it names, and
nobody would have looked. The defect is the short-circuit, not the colour it happened to
produce.

The tell is textual and greppable: belt-and-braces language in a Setup — "as a second
safety net", "just in case", "also blocked so it can't escape". A second guard is free only
when it sits **downstream** of the subject. Whenever the product's own branch order puts
the belt first, the net is a short-circuit.

**Rule.** For any scenario whose subject is a *late* branch, write the product's branch
order into the specification and mark which branch each setup choice lands on. Prefer the
single narrowest guard provably downstream of the subject over stacking guards upstream —
in the case above, the sending-domain fence, which rejects locally *after* the send begins,
was sufficient alone and was already present. When auditing, ask **"what does my setup
prevent?"** — the mirror of B9's "what does my setup publish?"

**Cost.** The author must read the handler's control flow, not just its contract. B9 costs
you the provisioning path; this costs you the branch order.

## C. Verdict laundering

### C1 — Re-run until green

**Symptom.** The suite is green. A regression shipped.

**Mechanism.** A scenario fails. It looks like flakiness. Someone re-runs. It passes. The
failure is filed as a flake. It was real, and it was reproducible only some of the time.

**Rule.** Absorb delay with a **bounded poll**, never a retry. The only legitimate retry is
after `INFRA`, and it is counted ([`03-evidence-rules.md`](03-evidence-rules.md) R4,
[`02-verdicts.md`](02-verdicts.md)).

**Cost.** Genuine environment flakiness now needs diagnosing rather than dismissing.

### C2 — `BLOCKED` as a judgement call

**Symptom.** The blocked pile grows. Occasionally a regression is sitting in it.

**Mechanism.** An agent that cannot complete a scenario concludes the scenario was "not
applicable here." If `BLOCKED` is a judgement, it becomes the drain that failures escape
through.

**Rule.** `BLOCKED` is computed from the specification's declared `requires`, and its reason
is the name of the missing requirement ([`02-verdicts.md`](02-verdicts.md)).

**Cost.** Preconditions must be declared explicitly, per scenario.

### C3 — Documented-not-driven counted as passing

**Symptom.** The pass count includes scenarios that ran no assertion.

**Mechanism.** A scenario deliberately describes an undrivable behavior. Having no
assertion, it does not fail. Anything that does not fail drifts toward being counted as
passing.

**Rule.** `N/A` is declared in the specification and never inferred. Report it separately.
A scenario with an `expect` block is always a `PASS`/`FAIL` candidate
([`02-verdicts.md`](02-verdicts.md)).

**Cost.** Blind spots stay visible in your reporting. That is the intent.

### C4 — Re-planning a failure into a pass

**Symptom.** A scenario failed, was re-planned, and the run reports green.

**Mechanism.** Plan caching invalidates on assertion failure. If invalidation silently
triggers a re-plan and the re-planned scenario passes, the failure vanishes — and this is
re-run-until-green with better tooling.

**Rule.** Report the failure. Re-planning is a separate, explicit action whose outcome is
visible ([`11-economics.md`](11-economics.md)).

**Cost.** An extra step when a plan genuinely goes stale.

### C5 — `INFRA` laundering

**Symptom.** The suite is green. A regression shipped. No `FAIL` was ever recorded.

**Mechanism.** `INFRA` sanctions a retry, and the `FAIL`-versus-`INFRA` boundary is
"empty result versus probe error." If that boundary is classified by the executor rather
than mechanically, an accommodating agent — or a probe wrapper that turns an empty result
into a non-zero exit — classifies inconvenient emptiness as a probe error, earns a
legitimate retry, and eventually passes.

Re-run-until-green survives, laundered through the method's own retry rule. This is a false
green created by C1's own remedy.

**Rule.** `INFRA` requires a **recorded raw probe error** in the bundle; an `INFRA` without
one is a harness finding. Classify mechanically wherever possible — exit codes, transport
errors, unresolvable bindings. Cap retries. Report `PASS after INFRA retry` as its own count,
never folded into `PASS`. See [`02-verdicts.md`](02-verdicts.md#infra).

**Cost.** Probes must distinguish "ran and found nothing" from "failed to run," which means
real error handling rather than best-effort.

---

## D. Wrong target

### D1 — Another service answered

**Symptom.** A scenario passes. The service it was meant to exercise was never involved.

**Mechanism.** Systems with multiple entry points route by path or host. Send a request to
the wrong one and a *different* service answers — sometimes with a plausible response. A
health check is the classic case: a shared path returns a healthy response from whichever
service actually owns it.

**Rule.** The header table declares the surface; the binding maps surface to host; a
preflight at run start proves routing by requesting something only the intended service
serves ([`10-environments.md`](10-environments.md)).

**Cost.** A routing preflight per run, and a binding file.

### D2 — Reading one replica

**Symptom.** Intermittent reds on side-effect assertions. Re-running "fixes" them, which
feeds C1.

**Mechanism.** A request lands on one replica, so its log line exists on only that one.
Reading a single replica — or a command that silently picks one — misses the evidence a
predictable fraction of the time.

**Rule.** Always aggregate across replicas with a selector, not a captured instance name.
Check your tooling's tail defaults; aggregating modes sometimes truncate more aggressively
than single-instance modes ([`06-side-effects.md`](06-side-effects.md)).

**Cost.** Slightly more expensive queries.

### D3 — Asserting a response the system cannot produce

**Symptom.** The assertion passes, and the code path the author believed they were testing
has never once executed.

**Mechanism.** Distinct from plain under-assertion (E1), though they combine. The author
reads the source, finds a defensive branch — one that middleware, routing, or an earlier
guard makes unreachable — and writes assertions describing *that branch's* response. At
runtime a different component answers, from a different code path, with a response loose
enough to satisfy the assertion: the same status, a compatible error token.

The tell is that the scenario would still pass if the branch it names were deleted.

**Rule.** Assert what the system returns, confirmed by observing it at least once — never
what the source suggests it might. Where two components can produce the same status for a
scenario, assert something that distinguishes them (the exact token, a header, a body shape
only one of them emits).

**Cost.** Every assertion needs one confirmed observation before it is trusted, which is a
real cost when writing specifications ahead of implementation — tag those `@intent` and
confirm them at first run.

---

## E. Assertion weakness

### E1 — Under-assertion

**Symptom.** Everything passes. The values are wrong.

**Mechanism.** `body.status is string` where the value is deterministically `"ACTIVE"`. The
assertion is true for any string. This survives every other rule on this page: the
correlator is right, the evidence is real, the assertion evaluates true, and the check
proves nearly nothing.

**Rule.** Assert exact literals wherever the value is deterministic
([`03-evidence-rules.md`](03-evidence-rules.md)).

**Cost.** More assertions to maintain when the contract legitimately changes.

### E2 — Weakened after a copy edit

**Symptom.** An assertion that once verified a behavior now verifies almost nothing.

**Mechanism.** The assertion matched human-readable message text. Someone reworded the
message. The scenario failed for a non-defect. To stop the noise, the assertion was loosened
to a substring, then to presence, and now it is E1.

**Rule.** Assert stable machine-readable codes; treat message copy as loose corroboration
only. A changed stable code **should** fail — that is a breaking change
([`03-evidence-rules.md`](03-evidence-rules.md)).

**Cost.** Your API needs stable codes. It should have them anyway.

### E3 — The wrong envelope on a carve-out surface

**Symptom.** The assertion passes against a surface that does not speak the shape being
asserted.

**Mechanism.** Systems often have one first-party response envelope plus several protocol
surfaces that deliberately keep their own error shapes. Asserting the house envelope on a
protocol surface can pass by coincidence — a shared field name, a compatible status — while
verifying the wrong contract entirely.

**Rule.** The header table declares the wire shape, and it is **classified per handler, not
per file**. One source file routinely mixes both kinds ([`04-spec-altitude.md`](04-spec-altitude.md)).

**Cost.** Authors must know which surface they are on. They should.

### E4 — The oracle that computes its own expectation

**Symptom.** A precise, two-sided, well-reasoned assertion. It passes with the control
deleted.

**Mechanism.** When a constant expectation proves brittle — it depends on environment
speed, load, or timing (G9) — the natural repair is to **derive** the expectation at
runtime from something measured during the run. The assertion stops being a magic number
and starts being principled. It can also stop being able to fail.

The trap is that the measured input is frequently **itself downstream of the failure
mode**. Removing the control moves the measurement in the same direction it moves the
observation, and the two travel together: the defect funds its own alibi.

A per-connection rate limiter (burst 10, refill 1/sec) was asserted by flooding a socket
with 20 frames and requiring at least one rejection. That count turned out to depend on
server latency, so the proposed repair measured elapsed time `T` across the persisted rows
and asserted `|rows − (burst + rate·T)| ≤ 2` — parameterised by the real contract, immune
to environment speed, and wrong. Delete the limiter: all 20 frames are accepted, each costs
its full processing time, so `T` inflates to ≈9.5s, the expectation becomes
`min(20, 10 + 9.5) = 19.5`, and `|20 − 19.5| = 0.5` **passes**. The crude bound it replaced
— `rows < 20` — caught exactly that case. The repair was a downgrade that read as a strict
improvement, and it was two review passes from shipping.

**Rule.** **A measurement-derived oracle is falsifiable iff the measured input cannot
absorb the failure mode it targets.** Test that directly rather than reasoning about it:
compute the oracle's verdict under the mutation it exists to catch, on paper, before
shipping it. If the answer is PASS, the oracle is decorative. Three properties restore it:

1. Pin the **contract** constants as specification literals; measure only the free variable.
2. Size the stimulus so the derived bound sits far from the trivial upper limit — if
   `expected` can approach the total sent, "everything was accepted" is inside the band.
3. Where no sizing works, gate the region rather than asserting into it (G9).

**Cost.** Every derived oracle needs an explicit refutation check against its own target
mutation, recorded in the specification beside the derivation.

---

## F. Harness rot

### F1 — The harness calls a dead path

**Symptom.** Setup silently stops working. Scenarios fail for reasons unrelated to the
behavior, or fixtures are quietly not created.

**Mechanism.** A contract changed — a method, a path, a parameter location. The
specifications were updated. The **runner scripts and the prose** were not, because the
change swept only the obvious directory.

**Rule.** Make as much of it mechanical as possible, and be honest that the remainder is
not. Gateable today: every route in the API schema has a specification; every declared code
appears in the published manifest; every header-table field matches deployed configuration;
no specification or runner references a path absent from the schema. **Not gateable:**
whether a *runner script* still uses a live contract — that is code, and only running it
proves it.

So the real rule is narrower than "sweep everything": **anything the runner does during
setup must be exercised by at least one scenario's assertions.** A setup path nothing
asserts on can rot silently, and the fixture it produces will be quietly wrong rather than
absent.

**Cost.** Some coverage exists only to keep setup honest. That is a legitimate use of a
scenario.

### F2 — Tooling fails silently

**Symptom.** An entire class of evidence acquisition stops working. Scenarios depending on
it degrade in whatever direction their assertions point.

**Mechanism.** A helper depends on a tool version, shell feature, or environment assumption
that is not present. It errors on a path nothing checks, and the caller carries on.

**Rule.** Evidence acquisition fails **loudly**, as `INFRA`. A probe that returns nothing
because it crashed must be distinguishable from one that returns nothing because there is
nothing there — those are `INFRA` and `FAIL`, and collapsing them hides the first
([`02-verdicts.md`](02-verdicts.md)).

**Cost.** Probes need real error handling rather than best-effort.

### F3 — A stale header table

**Symptom.** No visible symptom. The specification asserts a contract the system no longer
has.

**Mechanism.** The rate-limit tier, auth requirement, or surface changed. The table did
not. Nothing checks it, so nothing says so.

**Rule.** Header-table fields are contract, and a stale one is silent drift. Gate them
statically against deployed configuration ([`09-gates.md`](09-gates.md)).

**Cost.** The gate needs access to the deployed configuration values.

### F4 — A duplicate stimulus that is a no-op

**Symptom.** A side-effect assertion fails mysteriously, or an absence assertion passes
falsely.

**Mechanism.** Many systems deduplicate. Re-using an identifier, or acting on a subject a
previous run left behind, means the stimulus is correctly ignored — no new effect is
produced. Depending on which way the assertion points, that is a confusing red or a
comfortable green.

**Rule.** Mint a fresh identifier per execution. Where a scenario requires a subject to be
absent, **assert it is absent first** rather than assuming
([`07-isolation.md`](07-isolation.md)).

**Cost.** An extra precondition check on scenarios that create things.

### F5 — The check passes on an artifact from a previous run

**Symptom.** A preflight or acquisition step reports success and exits 0. It did not
succeed. The evidence it validated was written by an earlier run.

**Mechanism.** Distinct from F2, and the distinction is the whole point: F2 fails
*silently* — the probe returns nothing and the caller carries on. This one fails
**loudly and is then overruled**. The acquisition step prints its error; the script does not
abort (no `set -e`, or a helper that `return`s non-zero into a caller that ignores it); the
assertion that follows reads a **persisted artifact** — a token file, a captured id, a
cached manifest — left on disk by the last successful run, and finds it perfectly
well-formed. So the run ends green on evidence it did not gather.

Persistence is the mechanism. Anything the harness writes to a stable path and re-reads
later can substitute for a failure: session tokens, `.run/` scratch, a downloaded fixture, a
generated config. The assertion is usually correct in isolation — it checks a real property
of a real artifact ("the token's role claim is admin") — and the property is *stable across
runs*, which is exactly what makes the stale copy pass.

What happens next depends on luck, not design. If the stale artifact has expired, downstream
scenarios fail and get attributed to the product — expensive, but visible. If it is still
valid, the suite runs to green against a **previous session's identity** and the broken
acquisition is never discovered. We observed the first; the second is the same bug on a
shorter clock.

**Rule.** Acquisition failure is fatal — make it impossible for the assertion to run at all.
Then assert **provenance, not just content**: that the artifact was produced by *this*
execution (mtime newer than run start, a run id written into it, or simply deleting it
before acquiring). A check on a persisted artifact that cannot distinguish "fresh" from
"left over" is not a check. When porting a working script, re-verify its error handling at
the destination: this arrived by copying a correct original into two repos and losing the
hard exit on the way.

**Cost.** Explicit failure propagation in shell helpers, and one freshness assertion per
persisted artifact.

---

## G. Structural

### G1 — Intended non-event with no positive control

**Symptom.** An "this must not fire" scenario passes forever, including after the
publishing pipeline breaks entirely.

**Mechanism.** The scenario polls for absence. Absence is equally well explained by "the
feature works" and "the observation path is dead." With no control, those are
indistinguishable, and one of them is permanent green.

**Rule.** Inverted verdicts require three things
([`06-side-effects.md`](06-side-effects.md#intended-non-events)): a **per-execution-unique
subject**, a **broad** predicate over a minted marker (breadth inverts for absence — a
narrow pin creates misses, not safety), and a **positive control of the correct kind**.

On the control: an earlier version of this rule required "the identical predicate," which is
**impossible** for the most common case. In an authorization or tenancy scenario the target
event can only be produced by violating the property under test, so no legitimate action
produces a matching event. Two kinds exist and they are not interchangeable:

- **Same-subject control** — the subject may legitimately emit this event type. Identical
  predicate; capture and drain; count-of-survivors applies.
- **Proxy control** — the target event is producible only by violating the property. The
  predicate necessarily differs, there is nothing to drain, count-of-survivors does **not**
  apply, and the residual gap is that routing for the target key stays unproven. State that
  gap in the specification and route the control through every routing-relevant component.

Bracket the control — run it again after the absence poll — or state the residual window.
A control only at the front proves the pipeline was live when the window opened.

**Cost.** Inverted scenarios become more elaborate. They are also the ones most likely to be
silently wrong, and a wrong one passes forever.

### G2 — A regression net believing it is an oracle

**Symptom.** A full green run against a system that never did what the requirement asked.

**Mechanism.** Every assertion was derived by reading the implementation. The specification
faithfully records the implementation's misunderstanding, and green means "still doing what
it did," which was never right.

**Rule.** Tag provenance per assertion. Report the ratio. Ensure the behaviors you would
least like to be wrong carry requirement-derived assertions
([`05-provenance.md`](05-provenance.md)).

**Cost.** Provenance discipline, and the discomfort of publishing a low ratio.

### G3 — Unauditable greens

**Symptom.** Nobody can answer whether a scenario was genuinely green last month.

**Mechanism.** Only summaries were kept. The evidence is gone, so a past `PASS` is an
unverifiable claim — including the claim that its assertions were evaluated at all.

**Rule.** Archive evidence bundles. A verdict must be recomputable from its bundle without
re-running anything ([`08-evidence-bundles.md`](08-evidence-bundles.md)).

**Cost.** Storage, and redaction discipline.

### G4 — Coverage mistaken for verification

**Symptom.** Every endpoint has a specification. Defects ship regularly.

**Mechanism.** Coverage counts enumeration, not assertion quality. A suite asserting
`status == 200` everywhere reports total coverage and is E1 at scale.

**Rule.** Do not gate on a coverage percentage. Prefer the provenance ratio and
assertion-per-scenario density as health signals, and do not turn those into thresholds
either ([`09-gates.md`](09-gates.md)).

**Cost.** No single number to report upward. There was never an honest one.

### G5 — Configuration divergence between environments

**Symptom.** A full green run in the verification environment. The behavior is broken in the
next environment, and the suite said nothing.

**Mechanism.** Verification transfers between environments only as far as configuration
matches. A feature flag on in staging and off in production, a different limit, a different
timeout, a disabled integration — the scenario exercised a code path that the target
environment does not take. Nothing in the suite notices, because nothing in the suite knows
what configuration it assumed.

**Rule.** Declare the configuration the corpus depends on in the environment file, assert it
at run start like the routing preflight, and diff configuration across environments before
promoting. Every divergent key is a scenario whose result does not transfer — name it in the
run ledger's "not run" section rather than assuming
([`10-environments.md`](10-environments.md)).

**Cost.** Someone has to enumerate the configuration the suite depends on, and keep it
current.

### G6 — Provenance inflation

**Symptom.** The `@intent` ratio rises over time. Requirement coverage does not.

**Mechanism.** A tag is written once; the assertion it describes is edited many times. An
`@intent` assertion goes red, someone updates the expected value to match observed behavior,
and the tag stays. The assertion is now implementation-derived and still claims otherwise.
The gate checks tag *presence*, so nothing catches it, and the drift is always in the
flattering direction.

**Rule.** `@intent` requires a resolvable citation or fails the gate; editing an `@intent`
expected value requires re-citation or downgrade to `@contract`; re-adjudicate the top-ten
behaviors quarterly ([`05-provenance.md`](05-provenance.md)).

**Cost.** Provenance becomes a thing you maintain rather than a thing you declared once.

---

### G7 — A universal claim over an empty set

**Symptom.** A scenario named for a specific leak or invariant — "the list never returns
the raw secret", "no message from another tenant appears" — is green, has always been
green, and would stay green after the leak ships.

**Mechanism.** The assertion is universally quantified: *no element has property P*. Every
such claim is **vacuously true when the collection is empty**, and nothing in the scenario
establishes that it is not. The assertion can be as strong as you like — exact literals,
stable codes, the whole of E1's discipline satisfied — and still range over zero elements,
because assertion *strength* and quantifier *domain* are independent.

The domain is usually emptied by something outside the scenario: a sibling's teardown
revoked the only record, the fixture is created later in file order, or the seed was
tenant-scoped to a tenant with nothing in it. So the scenario was correct when written and
became vacuous when a neighbour changed — with no edit to the scenario itself and no
symptom.

Two aggravating shapes, both observed in one corpus:

- The claim written as **prose in a parenthetical** — "(each element carries `key_prefix`;
  no element carries `key`)" — sitting beside an `is array` assertion. It reads like an
  obligation and is evaluated by nothing.
- The claim naming a record **a prior scenario's teardown deletes**, so the domain is
  guaranteed empty by construction, permanently.

This is G1's disease on a different axis. G1 is absence over *time* with no proof the
observation path was live; this is absence over a *set* with no proof the set was
non-empty. The remedy has the same shape — a control — but the trigger is different, and so
is the detection question. For G1 you ask "what proves the pipeline was up?" For this you
ask **"how many elements does this quantifier range over, and what guarantees that?"**

**Rule.** A universal claim requires a **witness**: at least one element that *would*
exhibit P if the defect were present, minted by the scenario itself rather than inherited.
Assert the witness is present *before* asserting the property is absent from it. Where the
witness must be a real credential or a real cross-tenant record, that makes the scenario
`@destructive` and it needs teardown — pay that, or the scenario is decoration.

Corollary for reviewers: an assertion of the form *no X has P* is not reviewable without
knowing the cardinality of X at that instant. Treat "the list is empty here" as a **FAIL**
of the scenario's premise, not as a pass.

**Cost.** Witness setup and teardown on exactly the scenarios that are currently cheapest,
which is why they were written this way.

### G8 — A completeness claim over a silently truncated search space

**Symptom.** An audit reports "no remaining occurrences", "all call sites updated", "nothing
left to migrate" — and the count is wrong, because the tool that produced it never looked at
part of the corpus and did not say so.

**Mechanism.** G7 is a quantifier ranging over an **empty** set. This is a quantifier ranging
over a set that is silently **smaller than the one being claimed about**. The claim is
"across the repository, zero matches"; the evidence is "across the subset my tool chose to
read, zero matches"; nothing in the output distinguishes the two, because a search that finds
nothing and a search that looked nowhere print the same thing: nothing.

The exclusions are almost always *someone else's* sensible default, invisible at the call
site. A search wrapper that respects ignore files skips generated, vendored, and — critically
— **ignored-but-present** working files. A linter honours inline suppressions. A coverage tool
excludes a directory configured years ago. A test runner silently skips a suite whose fixture
is missing. In each case the tool is behaving correctly and reporting honestly about a domain
the caller never specified.

Observed: a rename audit reported one remaining reference across six repositories. A second
pass with a tool that had no ignore-file awareness found more than twenty, in files that were
present, tracked by the task, and read by the runtime — they were merely *gitignored*, which
the search treated as "does not exist". The first number was produced by a command that
exited 0 and looked exhaustive.

This is the failure mode of every *audit* rather than every *assertion*, so it escapes review:
reviewers check what the scenario asserts, not what the tool that verified the migration was
allowed to see.

**Rule.** A completeness claim must state its **domain and its exclusions**, and the tool must
be one whose exclusions you chose. Prefer a search that cannot skip (`/usr/bin/grep`, an
explicit file walk) over a convenience wrapper, and **calibrate it**: plant a known match in a
location you expect to be excluded and confirm the tool reports it. If you cannot say what the
search excluded, you have a sample, not an audit. Where a zero result is load-bearing, report
the **denominator** alongside it — "0 matches across 1,917 files" is auditable, "0 matches" is
not.

**Cost.** Slower exhaustive searches, and one calibration probe per audit tool — paid once,
not per audit.

### G9 — The verdict is a function of an undeclared environmental variable

**Symptom.** The same scenario, the same input, the same build, two runs, two different
numbers. Both recorded `PASS`, and neither run was wrong.

**Mechanism.** G5 covers *configuration* divergence — discrete keys someone can enumerate
and diff. This is its continuous twin: a performance property no configuration file records
(per-request latency, queue depth, replica placement, clock skew) that the scenario's
arithmetic silently depends on.

Such a scenario is usually derived correctly for **one** value of that variable, and the
derivation is written down — which is what makes it convincing. What is not written down is
that the variable is free.

A rate-limit flood asserted "at least one rejection" from a client sending at a fixed 250ms
cadence. But the server dispatched frames serially and inline, so the real spacing between
limiter decisions was `max(client cadence, server processing)` — client-bound on a fast
day, server-bound on a slow one. Two runs on identical input produced 6 rejections/14 rows
and 1 rejection/19 rows. Solving for the boundary, **zero** rejections occur once
processing reaches 0.53s; the observed run sat at 0.50. The scenario was ~5% of an
unrelated latency drift away from a `FAIL` that would have been filed against a rate
limiter working exactly as designed.

"5% of margin" is the flattering framing. The variable is not a stable property but the
mean of a noisy distribution, so the verdict was a function of a random variable straddling
a cliff.

There is often a **hard floor** underneath, and it is worth finding: past some value of the
variable, the presence and the absence of the control become *observationally identical*,
and no assertion at any sample size separates them. Below the cliff you have a scenario;
above it you have a coin.

**Rule.** When a scenario's expected value is derived, state the environmental variable it
is derived **against** and solve for the value at which the verdict flips. If that value is
reachable in the target environment, either resize the stimulus until the flip point is far
outside the plausible range, or declare the unobservable region as a precondition and
report `BLOCKED` inside it ([`02-verdicts.md`](02-verdicts.md)) — never `PASS`, never
`FAIL`.

**Cost.** One piece of arithmetic per derived scenario, and an environmental threshold that
must be re-derived when the workload changes.

### G10 — The specification documented the defect as correct

**Symptom.** A live defect, a suite that covers the exact behavior, and a specification
that describes the defect accurately — as the contract.

**Mechanism.** Every other entry in this catalogue describes a check that cannot fail. This
one describes a check that **would** have failed and was corrected *toward* the bug.

The path is ordinary and feels like diligence. A scenario is written. A run produces an
unexpected observation. The author investigates, constructs a plausible mechanism for why
the observation is legitimate, and writes it into the specification as a Note — often with
a table, often with an instruction not to change it. The suite is then permanently blind,
and the blindness is *documented*, which makes it far more durable than a weak assertion:
the next author who notices the anomaly finds a prior explanation and moves on.

A broadcast transport delivered duplicate frames whenever the publisher and subscriber were
the same node. The specification recorded a placement table stating that two frames were
"legal on this transport — do not fix it", and relaxed the count assertion to `>= 1`. The
mechanism offered was real: the transport genuinely has at-most-once *arrival* semantics.
It simply did not license the duplicate, which came from a missing self-filter. The product
shipped the duplicate for months behind an accurate description of it.

The diagnostic is **provenance, not plausibility**: was this explanation derived from the
source, or from the observation? An explanation reverse-engineered to fit an observation
will nearly always *be* plausible — that is the property it was constructed for
([`05-provenance.md`](05-provenance.md)).

**Rule.** A specification's expected values come from the contract — the code, the schema,
the declared behavior. When a run contradicts the specification, the permitted outcomes are
"the product is wrong" (file it) and "the contract is other than I believed" (**cite the
contract**, not the run). An observation may never be its own justification. In review,
treat every Note explaining why an observed value is acceptable as a diff-time question:
*what is the citation?* Prose arguing an anomaly is fine, with no reference to the source,
is the signature.

**Cost.** Anomalies cost a source reading rather than an inference, and some stay open
longer.

## H — Omission (the assertion nobody wrote)

Every entry above describes an assertion that was written and is wrong. This section is
different: the assertion is **absent**, the suite is coherent, and no rule that reads the
corpus can object — because there is nothing there to read. Both entries were found in a
live trial of this method, in corpora that obeyed every other rule in this catalogue.

They are the hardest class to find and the easiest to argue away, because the suite looks
*good*. That is the point.

### H1 — Exactly-once asserted by existence

**Symptom.** A duplicate event, notification, refund, or webhook ships. The suite is green
and stays green. Every assertion about the event is correct as written.

**Mechanism.** The obligation is *exactly one*. The author writes `published … exists`, or
`count == 1 within`. Existence is true of one and equally true of five. A first-true-wins
`count == 1` is satisfied the moment the count reaches one and stops looking, so it catches
only a duplicate already in flight before the first poll — never a retry or a redelivery,
which is where duplicates actually come from.

The B6 rule does not fire, because B6 polices `count` assertions and no count was written.
The gate is keyed on a syntactic form; the defect is a missing obligation. **The rule and the
defect never meet.**

Observed shape from the trial: the same author applied cardinality discipline correctly **28
times** to persisted state and **zero** times to events, mirroring the distribution of
examples in the method itself — every `count`/`hold` exemplar was store-flavoured and the
canonical event exemplar was existence-shaped. The method did not merely fail to catch this;
its examples taught it.

**Rule.** *If the intent is "exactly N", no assertion form satisfiable by "at least N" is
sufficient.* Use the **receipt plus bracket** pair — `count == n within` *and* a `hold` /
`absent through` over the retry-or-redelivery interval
([`06-side-effects.md`](06-side-effects.md)). Gate it with the scope qualifiers in
[`09-gates.md`](09-gates.md), and generate the obligation from the event catalogue rather
than hoping an author reaches for it.

**Cost.** Exactly-once scenarios take the bracket's full budget instead of returning on
first sight, and someone must write down which events are exactly-once.

### H2 — The boundary chosen by judgment

**Symptom.** A 5xx reachable in one request, often unauthenticated, sitting behind an input
bound the suite believed it had probed.

**Mechanism.** The author probes "a large value" rather than *the* boundary. In the trial,
three independent suites hunted this class and picked `2147483000`, `999999999`, and
`2147483647` — respectively 647 below the boundary, far below it, and exactly *on* the last
good value. One suite of three landed in the failing window. The parsing layer was 64-bit and
the storage column 32-bit, so the live window sat **between two declarations**, and no amount
of "pick a big number" finds it reliably.

Judgment-picked boundaries fail in the flattering direction: the probe returns the expected
error, the scenario passes, and the author records the bound as verified.

**Rule.** Derive boundary probes from the **declared type**, not from judgment — and from the
**narrowest** declaration in the path, since a validating parse and a storage column may
differ. Mechanically: `max`, `max+1`, `min−1`, and the storage-width boundary. This is a
completeness obligation, generated from the schema, not a check on what the corpus says
([`09-gates.md`](09-gates.md)).

**Cost.** You need the schema, and you need to know which declaration is narrowest.

---

## Adding to this catalogue

Use the format above, and be precise about the mechanism. "The agent got confused" is not a
mechanism. "The correlator was empty and the agent matched on the message string, which
matched a sibling scenario's log line" is.

The mechanism is what makes an entry actionable, because it is what tells the next reader
whether their suite has the same hole.
