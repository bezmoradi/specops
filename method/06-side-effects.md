# 06 — Side effects

The response is the easy half. Most of what a system does that matters happens after it
answers: a row is written, an event is published, a consumer acts on it, a message is
delivered, a counter moves.

Verifying that half is where agent-executed specification testing earns its cost, and where
it goes wrong.

## The independence rule

**The response and the side effect are independent verdicts. Both must pass.**

A `2xx` is not evidence that a downstream effect occurred. Systems return `202 Accepted`
and then drop the work. Systems commit a row and fail to publish. Systems publish to the
wrong topic. Every one of those returns a perfectly good response.

## Correlator-first: ranking evidence

The instinct is to rank evidence by how *stable* it is — a metric name is more stable than
a log message, so prefer the metric. In a shared environment that instinct is wrong.

**Rank evidence by whether it can carry a correlator unique to your execution.**

| Tier | Evidence | Why |
|---|---|---|
| **Authoritative** | Persisted **state** — a row that exists, is gone, or changed | The strongest signal available. It proves the outcome, not that a handler ran. |
| **Authoritative** | A **log line or event** carrying your correlator | Isolable to your execution; immune to concurrent activity. |
| **Corroboration only** | A **counter delta** | Structurally cannot carry a correlator. |

Metric attributes are deliberately low-cardinality — that is what makes metrics affordable
— so there is no room for a per-execution identifier, and your `+1` shares a bucket with
every other occurrence in the window.

- **Assert state and correlated evidence as the verdict.**
- **Assert counters as `moved`, never an exact delta.**
- **Exception:** a counter keyed by an attribute nothing else emits is isolable and may be
  asserted exactly.

**Store observation is a sanctioned evidence channel, not an escape hatch.** Reading
persisted state to confirm an outcome is the top tier above. Writing state the product
cannot produce is the exception, defined below. The two are separate concepts with separate
rules — see [`04-spec-altitude.md`](04-spec-altitude.md) for the distinction and the
persistence carve-out on the refactor test.

## Choosing a correlator

Unique to this execution, and present in the evidence. In order of preference:

1. **A minted identifier you control** — when the stimulus is an event publish, the suite
   generates the `event_id`.
2. **A propagated trace identifier** — downstream evidence carries the same trace as the
   originating request.
3. **A freshly created resource id.**
4. **A per-run unique subject** — an address, name, or reference nothing else touches.

**Never a time window.** "The evidence that appeared in the last thirty seconds" collides
the moment two scenarios run concurrently, and it collides silently.

**A freshness bound is not a time correlator.** Requiring evidence to postdate the recorded
stimulus is a *filter* that composes with a correlator. It is never a substitute for one,
and it is specifically required on retries ([`12-false-greens.md`](12-false-greens.md) B5).

Note an asymmetry in tier 2: some systems deliberately do not log certain fields — personal
data, for instance. If the identifier you would naturally search by is never emitted, the
search returns nothing forever, and fail-closed turns that into a permanent red that looks
like a product failure. Find the identifier the system *does* emit and record the bridge in
your execution contract.

## Chained evidence, and per-hop budgets

```
create ──► [hop 1] read the index to resolve the new id
       ──► [hop 2] find the originating evidence, extract the trace
       ──► [hop 3] assert the downstream consumer's evidence for that trace
```

**Every hop needs its own budget.** A single budget on the last hop hides which one was
slow, and an unbounded early hop produces an empty capture — which R3 correctly turns into
a `FAIL` that looks like a product defect but is not.

## Appearance versus invariant

The most common side-effect error is asserting an invariant with appearance semantics.

- `within <budget>` polls until true; **the first true evaluation wins**.
- `hold <budget>` samples throughout and requires the condition to be true at every sample
  and at budget expiry.

Anything shaped "still equals", "exactly N", or "unchanged" **must** use `hold`. Written
with `within`, it is satisfied the instant it is first true — which for a
count-that-must-not-grow is usually before the system could have done the wrong thing.

`hold` alone does not prove the stimulus arrived. Pair it with evidence of receipt — a
dedup-hit log line, a processed counter, a consumer acknowledgement. Otherwise
zero-delivery satisfies the invariant trivially.

## Exactly-once: the receipt is half an assertion

A stimulus that should produce **one** of something — one event, one notification, one
refund, one webhook — needs **two** assertions, and writing only the first is the most
expensive mistake on this page. It was found in a live trial of this method, in which a
double-publish defect survived a corpus that applied every other rule here correctly.

```
✗   event(billing).published where order_id == {{ORDER_ID}}   within cross_service
        exists

✗   event(billing).published where order_id == {{ORDER_ID}}   within cross_service
        count == 1

✓   event(billing).published where order_id == {{ORDER_ID}}   within cross_service
        count == 1
    event(billing).published where order_id == {{ORDER_ID}}   hold single_hop
        count == 1
```

**Why the first is wrong.** `exists` is true of one message and equally true of five. An
existence assertion cannot express "exactly one" and never fails on a duplicate. This is the
form authors reach for, because it reads like the obvious thing to check.

**Why the second is wrong, and this is the subtle one.** `count == 1` looks like it fixes
it, and under the receipt carve-out it correctly takes `within` — the subject did not exist
before the stimulus, so this is an appearance claim, not an invariant. But `within` is
first-true-wins: it is satisfied the moment the count reaches 1, and it *stops looking*. A
duplicate that lands a millisecond later, or on a retry, or on a redelivery, arrives after
the assertion has already been satisfied and passes forever.

It only catches a *simultaneous* duplicate — one where both messages are already in flight
before the first poll. That is the easy half of the defect class, and the half least likely
to reach production, because the interesting duplicates come from retry and redelivery paths
that are by definition delayed.

**The pair.** A receipt bounds the count *upward from nothing*; a bracket bounds it
*forward in time*.

| Assertion | Wrapper | Answers |
|---|---|---|
| **Receipt** | `within` | did it happen, and was it one? |
| **Bracket** | `hold` (or `absent through` on a second marker) | is it *still* one when the window closes? |

Both are required. Neither alone is an exactly-once claim, and the bracket's budget should
cover the system's retry or redelivery interval — if you do not know that interval, that is
itself worth discovering, because it is how long a duplicate has to arrive in.

**The general rule, which is not about counts at all:** *if the specification's intent is
"exactly N", no assertion form that can be satisfied by "at least N" is sufficient.*
`exists`, `count >= n`, and a first-true-wins `count == n` all fall on the wrong side of
that line. [`09-gates.md`](09-gates.md) gives the mechanical form; the reason it is stated
here as intent rather than as syntax is that the trial's miss was an author holding the
right intent and reaching for a form the rule did not police.

## Stale reads on "unchanged" assertions

A survival or unchanged assertion evaluated through a cache or a read replica can return
pre-mutation state, so a destructive bug passes inside the staleness window.

For any `hold`, `exists`, or `unchanged` assertion:

- read a **strongly consistent** path where one exists, or
- **wait out the staleness bound** before the first sample, and say so in the specification,
  or
- prefer a **product read** over a store read — the product's own consistency guarantee is
  usually stronger and the altitude is better.

## The private sink pattern

The most robust way to verify "the right event was published" is to subscribe your own
private consumer to the topic, alongside the real ones. Fan-out gives every subscriber its
own copy, so the suite reads **the event as published** without racing the real consumer
and without depending on another service's log text.

Four disciplines. The last three exist because parallel lanes share one sink: every lane
reads the same queue, and a queue that leases each received message to one reader for a
while behaves differently under concurrent readers than under one.

- **Pin a per-execution-unique field.** A predicate matching only on event type will match
  a *previous* execution's lingering message for as long as the topic retains it.
- **Floor every read at the server's clock, taken just before the stimulus.** Only evidence
  stamped at or after the floor may match. This is the freshness bound above, applied to
  the sink; it composes with the correlator and never replaces it. Take the floor from the
  clock that stamps the events — the `Date` header of one of the service's own responses,
  for instance — never the harness's clock, whose skew moves the floor silently. Where setup
  may have emitted a same-shaped event, let that clock move more than one stamp granularity
  past setup before reading the floor; for whole-second stamps, two seconds also absorbs
  skew between replicas. Without the floor, the setup's own event
  ([`12`](12-false-greens.md) B9) or an earlier attempt's copy (B5) satisfies the correlator
  and is returned first — silently.
- **Never make a verdict depend on a delete.** A leasing queue honours a delete only when it
  carries the delivery token of the *latest* receive. Once a sibling has received the same
  message, your delete may do nothing and still report success. "Drain the setup's event,
  then assert on the next one" is therefore not an operation under concurrent readers: the
  drained message is still there to be matched. A capture may delete its own match as
  housekeeping. Noticing: search the corpus for a drain that precedes an assertion — each
  one is a floor that has not been written yet.
- **Survey by rotating a short lease, never by a zero-lease peek.** A receive that leases
  nothing keeps returning the front of the queue, so once the queue is deep a survey
  plateaus below the queue depth however long it polls. Receive with a lease of a few
  seconds and do not delete: what was just read is hidden, the next receive returns
  different messages, and siblings see every message again when the lease lapses. Record
  distinct messages surveyed against the queue depth when the poll started. A presence poll
  that stops short reports a false `FAIL` — loud. An absence poll that stops short has not
  observed absence: it is `FAIL` with that reason, never `PASS`, because a pass there is
  silent.

**A sink capture proves publish, not delivery.** Where the point is that the consumer acted,
assert the consumer's observable outcome too.

## Multiple replicas

**If a service runs N replicas, a request lands on exactly one, and its log line exists on
only that one.** Reading a single replica — or a command that silently picks one — misses
the evidence a predictable fraction of the time, and fail-closed turns each miss into a red
that is not real. The team learns to re-run, and re-running is the anti-pattern.

Always aggregate with a selector, never a captured instance name (which carries a
deployment hash and goes stale on the next rollout). Check your tooling's tail defaults;
aggregating modes sometimes truncate more aggressively than single-instance modes.

## Escape hatches

**An escape hatch is a WRITE.** Manufacturing state the product cannot be made to produce:
an expired token, a specific counter value, a corrupted row, another tenant's data.

Reading persisted state as evidence is not an escape hatch — it is the top evidence tier
above.

Four conditions on writes:

1. **Labelled.** Every use visibly marked.
2. **Confined to `requires` and `teardown`.** Never used as evidence *of* product behavior.
   If the only proof a feature works is a row you wrote, you proved nothing.
3. **`mutable` environments only.** Enforced by the binding, not by discipline
   ([`10-environments.md`](10-environments.md)).
4. **Preferred alternative first.** If an endpoint reaches the state, use it.

### The seeded subject must be visible to the product

A seeded row can be invisible to the application — a secondary index the raw write did not
populate, a derived field, a cache that never learns about it. Then the product returns
"not found" for *everyone*, the negative assertion passes, and the protection under test
was never exercised.

**Any scenario that seeds a subject and then asserts a caller cannot reach it MUST first
prove the product itself can reach it**, using the legitimate owner's credential:

```
​```setup
# seed-visibility control — the seed must be real to the PRODUCT, not just to the store
request GET {{surface(api)}}/v1/orders/{{FOREIGN_ORDER_ID}}
        Authorization: Bearer {{TENANT_B_TOKEN}}
expect  status == 200
​```
```

Without this, the scenario cannot distinguish "isolation works" from "the seed does not
exist as far as the application is concerned." See
[`12-false-greens.md`](12-false-greens.md) A4.

## Intended non-events

Some scenarios must prove an event did **not** fire. This inverts the fail-closed default:
a match is `FAIL`, no match through the full budget is `PASS`. The inversion is sound only
with all three of the following.

### 1 — A subject unique to this execution

Provision a throwaway subject nothing else touches. Polling for absence on a shared subject
lets any sibling drop a matching event into your window; the usual "fix" is to broaden the
predicate, after which the scenario can never fail.

### 2 — Broad predicate, not narrow

**Predicate breadth inverts for absence.** For a presence assertion, a narrow pin prevents
aliasing. For an absence assertion, a narrow pin creates *misses*: a bug that publishes with
a null, absent, or differently-shaped field slips past the predicate and the scenario goes
green.

Because rule 1 gave you a subject nobody else touches, you can afford breadth. Mint a marker
into the stimulus and match it **anywhere in the event**:

```
✗   event(billing).published where customer_id == "any"        # narrow; misses a null
✓   event(billing).published where any_field contains {{MARKER}}
```

### 3 — A positive control, of the right kind

A control proves the observation path is live. Without one, "no event found" is equally
explained by "the feature works" and "the pipeline is dead" — and the scenario reports
`PASS` forever after the pipeline breaks.

**There are two kinds, and using the wrong one silently weakens the guarantee.**

| | **Same-subject control** | **Proxy control** |
|---|---|---|
| When | The subject can legitimately emit this event type | The target event is producible **only** by violating the property under test |
| Predicate | Identical to the absence predicate | Necessarily different |
| Mechanics | Produce → capture → **server-clock floor** past it → then act. Count-of-survivors (matches stamped at or after the floor): correct ⇒ 0, regression ⇒ 1 | Produce over a parallel path; **no floor needed** — the control event can never match the absence predicate |
| Proves | The exact predicate matches when it should | The pipeline is live end-to-end |
| Residual gap | None material | Routing for the *target key* is unproven — a dead partition or mis-scoped subscription filter passes both |

Most authorization and tenancy scenarios need a **proxy** control: the only way to produce
`order.cancelled` for order X is to cancel order X, which is the thing that must not happen.
The count-of-survivors framing does **not** apply there, and neither does flooring past the
control.

**When you use a proxy control, state the residual gap in the specification.** A proxy
control must traverse every routing-relevant component — same topic, same subscription, same
consumer, same sink — so that the only unproven thing is the key.

### Bracket the control

A control before the action proves the pipeline was live *at the start* of the window.
Pipeline death after the control and during the absence poll still false-greens.

Either **bracket** — run the control again after the absence poll and assert it still fires
— or state the residual window in the specification. Bracketing is cheap and is the default.

### Structure

```
setup:    provision a per-execution-unique subject; mint {{MARKER}}
control:  produce a real event over the control path; assert it fired
          (same-subject: capture, then floor past it · proxy: no floor needed)
request:  perform the should-be-silent action
expect:   event(...) where any_field contains {{MARKER}}   absent through <budget>
control:  re-assert the control path still fires        ← bracket
```

**Never invert on a shared subject with a narrow predicate.** That is the one shape this
method cannot make sound.
