# 05 — Provenance

> Is your suite an independent oracle, or a very good regression net?
>
> Both are valuable. A suite that cannot tell you which one it is will be trusted for the
> wrong reasons.

This page describes a practice most suites — including mature, effective ones — do not
follow. It is here because its absence is the largest gap between what specification-based
verification *claims* and what it usually *delivers*.

## The claim, and the problem with it

The motivating argument for this whole method goes:

> An agent writes the implementation and the tests. Both can encode the same
> misunderstanding of the requirement. So separate them: write the specification
> independently, and it becomes an oracle that catches the shared mistake.

Now look at how specifications actually get written in practice. The usual instruction to
the agent is some version of:

> Verify the exact status code and field names against the actual handler. Never invent a
> response code.

That instruction is *correct* — it prevents fabricated assertions, which are worse than no
assertions. But follow it and the resulting specification is **derived from the
implementation**. It records what the code does.

Which means: if the implementation misread the requirement, the specification faithfully
records the misreading, the suite goes green, and the exact failure the method exists to
prevent has occurred with a full green run behind it.

This is not a reason to abandon the method. Implementation-derived specifications are
genuinely valuable — they catch regressions, they survive refactors, they document
behavior, and they are far better than nothing. It *is* a reason to be precise about what
you have.

## Two kinds of assertion

| Kind | Derived from | Catches | Does not catch |
|---|---|---|---|
| **`@contract`** | Reading the implementation, or observing the running system | Drift from established behavior. Regressions. Breaking changes. | A behavior that was wrong from the first commit. |
| **`@intent`** | A requirement, ticket, design document, or conversation — written **before** or **independently of** the implementation | Implementation that does not match what was asked for | Nothing extra; it is a superset of the above |

Both belong in a suite. `@contract` assertions are cheaper, more numerous, and carry most
of the regression value. `@intent` assertions are the ones that make the oracle claim true.

## Tagging

Mark provenance at the scenario level, and at the assertion level where a scenario mixes
both.

```markdown
## OR3 — Rejecting a duplicate customer reference

**Provenance:** @intent — requirement PAY-118, "a customer may not hold two open orders"
**Behavior:** creating a second open order for the same customer is rejected.

​```expect response
status == 409                                    @intent
body.code == "ORDER_ALREADY_EXISTS"              @contract
body.data.existing_order_id exists               @contract
​```
```

Read that block carefully, because it is the normal case: the *rule* came from a
requirement, the *encoding* of it came from the implementation. `409` is what the
requirement implies. The specific token and the shape of the response payload were
learned by reading the code.

That is honest, and it is fine. What is not fine is presenting the whole block as though
it were derived from PAY-118.

## The ratio, and what it means

Count it. Report it. It is the single most informative number about your suite, and far
more meaningful than a coverage percentage.

```
scenarios: 412
  @intent  : 71   (17%)
  @contract: 341  (83%)
```

**A suite that is 100% `@contract` is a regression net.** Say so. It will catch drift
extremely well and will never catch a requirement implemented wrong. That is a legitimate
and useful thing to own — most test suites in existence are this and do not know it.

**A suite with a meaningful `@intent` fraction on its high-value paths is an oracle where
it counts.** You do not need 100%. You need `@intent` coverage on the behaviors where
getting the requirement wrong would be expensive: money movement, authorization, data
deletion, anything irreversible.

## Raising the ratio

Four practices, in increasing order of cost and value:

**1. Write the specification before the implementation.**
The cleanest fix and the one that makes the oracle claim structurally true. When picking
up a new feature, the first artifact is the specification, derived from the requirement,
with no implementation to read. The agent then implements against it. Everything written
this way is `@intent` by construction.

This composes directly with spec-driven development workflows: the design specification
says what to build, the SpecOps specification says how you will know it was built. They
are written in the same sitting, from the same requirement, before any code exists.

**2. Increase the distance between the two authors.**
Independence is a spectrum, not a switch, and the mechanism matters: an ambiguity in a
requirement gets resolved from whatever priors the author holds. Two sessions of one model
share priors — so the correlated-misreading case, the one this whole method exists for, can
survive session separation intact.

| Separation | Defeats | Survives |
|---|---|---|
| Same session, spec then code | Nothing meaningful | Everything |
| Separate sessions, same model | Context contamination, carried-over assumptions | **Shared priors — the correlated misreading** |
| Different model | Most shared priors | Ambiguity both families resolve the same way |
| Different human | Shared priors | A genuinely ambiguous requirement |
| **Values derived from the requirement's own numbers** | **The ambiguity itself** | A requirement that is simply wrong |

Only the last row is *structurally* independent — it does not rely on the author, it relies
on the requirement containing the answer. That is practice 3 below, and it is why practice
3 outranks this one.

Practical recommendation: separate sessions as the floor for everything; a **different model
or a human** for the ten behaviors you would least like to be wrong. Do not claim more
independence than the separation you actually used.

**3. Derive assertions from the requirement's numbers.**
Requirements contain testable specifics — limits, timeouts, retention periods, retry
counts, currency rounding, threshold values. These are exactly where implementations quietly
diverge, and exactly where an `@intent` assertion pays for itself. If the requirement says
"a session expires after 8 hours of inactivity," assert 8 hours from the requirement, not
whatever constant the code contains.

**4. Backfill `@intent` on the high-value paths.**
For an existing suite, do not attempt a full audit. Take the ten scenarios covering the
behaviors you would least like to be wrong. For each, find the original requirement, and
re-derive the assertions from it *without reading the implementation*. Where the
re-derivation disagrees with the existing assertion, you have found either a stale
requirement or a real defect. Both are worth the afternoon.

## Keeping provenance current — the tag rots faster than the assertion

A tag is written once. The assertion it describes is edited many times, and the edits move
in one direction.

The failure: an `@intent` assertion goes red. Someone updates the expected value to match
what the system does, and the tag stays `@intent`. The assertion is now
implementation-derived and still claims to be requirement-derived. The ratio — "the single
most informative number about your suite" — inflates in the flattering direction, which
makes it a false green of the meta-layer.

Three rules close it:

1. **`@intent` requires a citation.** A requirement, ticket, or design-document reference,
   inline. `@intent` without one **fails the static gate**
   ([`09-gates.md`](09-gates.md)) — the gate checks presence *and* form, not presence alone.
2. **Editing an `@intent` assertion's expected value requires re-citation or downgrade.**
   Either the requirement says the new value — update the citation — or it does not, in
   which case the tag becomes `@contract`. There is no third option, and the diff makes
   which one happened visible in review.
3. **Re-adjudicate quarterly.** For the top-ten behaviors, re-derive the value from the
   requirement without reading the implementation, and confirm the citation still says what
   the assertion says.

If you find yourself editing an `@intent` value and reaching for the requirement to justify
it after the fact, that is the moment the tag is lying. Downgrade it.

## Reviewing provenance

When reviewing a specification, ask the author one question:

> Where did this number come from?

For every status code, limit, timeout, token, and threshold. The answers cluster:

- *"From the requirement / the ticket / the design doc."* → `@intent`.
- *"From the code."* → `@contract`. Fine, and tag it.
- *"It seemed right."* → **Stop.** This is the dangerous third category: neither derived
  from intent nor verified against behavior. It will pass by coincidence or fail by
  coincidence, and either way it means nothing. Send it back.

That third category is the reason this page exists in the operational half of the method
rather than the philosophical half. Untraceable assertions are common, they look exactly
like good ones, and asking where the number came from is the only cheap way to find them.

## Honest reporting

Whatever your ratio, state it where the suite's results are presented. A run summary that
says

> 563 passed, 0 failed

is compatible with a suite that has never verified a single requirement. A run summary that
says

> 563 passed, 0 failed — 17% of scenarios are requirement-derived (`@intent`); the
> remainder verify established behavior

tells the reader what kind of confidence they just received. That distinction is the whole
contribution of this page.
