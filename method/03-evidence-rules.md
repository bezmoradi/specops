# 03 — Evidence rules

Six rules, then the grammar that makes them enforceable. Each rule exists because its
absence produces a suite that reports green while the system is broken — and each failure
it prevents is **silent**, which is why they are rules and not preferences.

Read the threat model in [`01-overview.md`](01-overview.md#threat-model) first if you have
not. It bounds what these rules can and cannot defend against, and several of them are
routinely over-read.

---

## R1 — The agent never votes

**The agent's output is evidence. The verdict is computed from that evidence against the
declared assertions.**

An agent asked "did this pass?" is being asked to be helpful about an ambiguous situation,
and it will be. Given a `201 Created` and no observable trace of the downstream email, it
will conclude the email probably got sent. Given a log query that returned nothing, it will
conclude the log was probably rotated. Each individual inference is reasonable. In
aggregate they are a suite that cannot fail.

The split:

```
agent   ──►  decides WHICH probe answers the question, and runs it
        ──►  emits {request, response, captures, probe_results[]}

rules   ──►  parse the declared assertions
        ──►  evaluate them against the recorded evidence
        ──►  emit the verdict
```

**What this preserves:** the agent still decides *how* to observe — the entire reason for
using an agent. Choosing between a queue, a log, a row, and a metric is judgement, and it
is judgement the agent is genuinely good at.

**What this removes:** the agent deciding whether what it found is good enough.

**What it does NOT remove:** the agent choosing *which* evidence to report. R1 defends
against a fallible agent, not a deceptive one. A bundle whose probe read the wrong source
recomputes perfectly, because recomputation checks internal consistency, not truth. The
partial defenses — machine-validating probe targets against environment bindings, plan
replay, bundle audit — are named in the threat model. **"There is no other path to green"
is true of the judgement step and not of the pipeline.** Do not quote it as if it were the
latter.

**Failure mode without it:** silent and total. A suite whose agent votes reports a stable,
high pass rate indefinitely, including through outages.

**Cost:** assertions must be machine-evaluatable, which constrains the grammar below.

---

## R2 — Fail closed

**Missing, empty, or errored evidence is `FAIL`. Never `PASS` by inference.**

1. **A `2xx` never excuses an absent side effect.** Response and side effect are
   independent verdicts. Both must pass.
2. **An empty query result is a `FAIL`, not an inconclusive.**

The asymmetry that makes this necessary: response assertions are deterministic by
construction — the response either had the field or it did not. Side-effect assertions
require the agent to *construct* a query, and a badly constructed query returns nothing. So
the failure mode of the side-effect half must default to `FAIL`, or every malformed query
becomes a pass.

**One boundary that must stay sharp:** *empty* is `FAIL`; *errored* is `INFRA`. That
distinction is a retry channel, so it must be classified mechanically wherever possible —
see [`02-verdicts.md`](02-verdicts.md#infra).

**Failure mode without it:** the suite goes green precisely when observability degrades.

**Cost:** ordinary propagation delay produces reds. Paid by R4, not by relaxing this.

---

## R3 — No empty captures, no weaker correlator

**Any variable an assertion depends on MUST be non-empty before that assertion is
evaluated. An unresolved capture is itself a `FAIL`.**

**And: the agent may never substitute a weaker correlator than the specification
declares.**

A correlator is what ties evidence to *your* execution rather than someone else's. When it
comes back empty, the tempting move is to fall back to something broader — a message
string, an event type, a timestamp window. That fallback finds *a* piece of matching
evidence. In a shared environment it is very likely someone else's.

```
declared:   log line where trace_id == {{TRACE_ID}}
fallback:   log line where message contains "payment captured"     ← forbidden
```

**Rule:** an empty correlator fails the scenario. It does not relax it.

**A freshness bound is not a time correlator.** Requiring evidence to postdate the recorded
stimulus is a *filter* that composes with a correlator; it is never a substitute for one.
This matters for retries — see [`12-false-greens.md`](12-false-greens.md) B5.

**Correlator breadth inverts for absence assertions.** For a presence assertion, a narrow
pin prevents aliasing. For an absence assertion, a narrow pin creates *misses*: a bug that
emits the event with a null or differently-shaped field slips past the predicate and the
scenario goes green. The sound shape for absence is **unique subject, broad predicate** —
see [`06-side-effects.md`](06-side-effects.md#intended-non-events).

**Failure mode without it:** silent aliasing, more reliable the more you parallelize.

**Cost:** correlators must be designed rather than improvised.

---

## R4 — Bounded poll, never re-run-until-green

**Asynchronous evidence is polled at a stated interval up to a stated budget. Not visible
yet, keep polling. Absent after the budget, `FAIL`.**

R2 makes absence a failure. Real systems have propagation delay. Without a poll, R2 turns
ordinary delay into constant spurious reds — and the human reflex to a spurious red is to
re-run.

**Re-run-until-green is exactly how a real regression walks through a fail-closed gate.**

So the budget absorbs the delay, not the retry:

```
event(billing).published where order_id == {{ORDER_ID}}   within cross_service
```

Budgets are **named classes**, resolved per environment
([`10-environments.md`](10-environments.md)). Never literal durations in a specification:
the same behavior needs 5 seconds against an idle environment and 30 against a busy one,
and editing hundreds of specifications to tune that is how teams end up adding retries
instead.

Budgets are **per hop**. A chain of asynchronous hops needs one on each, or you cannot tell
which hop was slow.

**The one legitimate retry** is after `INFRA`. It is counted, capped, and reported as its
own outcome — never folded into `PASS` ([`02-verdicts.md`](02-verdicts.md)).

**Failure mode without it:** either constant false reds that train the team to ignore the
suite, or an undisciplined retry policy that lets regressions through.

---

## R5 — Write at an altitude that survives refactoring

**A specification states observable behavior. It never names an internal symbol, file, or
line. A behavior-preserving refactor must leave every specification valid.**

Stated in full in [`04-spec-altitude.md`](04-spec-altitude.md), including the one carve-out
that applies to store-observed evidence.

**Failure mode without it:** loud rather than silent — a refactor breaks a hundred
specifications. But the cost compounds: a suite that breaks on every refactor gets updated
mechanically, and mechanical updates do not preserve intent.

---

## R6 — Every assertion declares its provenance

**An assertion derived from reading the implementation is a regression check. An assertion
derived from a requirement is an oracle. Mark which is which, and keep the mark current.**

Stated in full in [`05-provenance.md`](05-provenance.md).

**Failure mode without it:** a suite that believes it is verifying intent while verifying
"the system still does what it did last month."

**Provenance has a second axis — whether the expected *value* was ever observed, or only
read out of the source.** Also in [`05-provenance.md`](05-provenance.md), and it is the one
that decides whether a brand-new scenario can be trusted on its first run.

## R7 — Assert the premise, not only the outcome

**A scenario whose setup silently fails to establish what it claims will report a product
defect. Assert the precondition inside the scenario so the failure is attributed to the
fixture instead.**

A `requires` block establishes that a fixture *exists*. It does not establish that the
fixture has the *property* the scenario depends on. That gap is where misattribution lives:
the setup runs, everything appears provisioned, and the scenario tests something other than
what it says.

```
# Scenario: a permission above the caller's ceiling is refused.
assert  catalog[{{PERM_ID}}].delegatable == false   # PREMISE — else this proves nothing
assert  status == 403
assert  body.code == "PERMISSION_ABOVE_CEILING"
```

Without the first line, a fixture that turns out to be *delegatable* produces a `200`, and
the scenario reports that the ceiling is broken. With it, the same run reports that the
fixture was wrong — which is true, cheap to fix, and does not send anyone to read
authorization code.

This is not hypothetical arithmetic. In one measured run of a freshly authored corpus,
**14 of 61 failures were preconditions that were never established**, each presenting as a
product failure. A second corpus on the same service carried premise assertions and caught
its one occurrence correctly on the first try.

**Where to use it:** any scenario whose meaning depends on a property of seeded data — a
role that must lack a permission, an account that must be unverified, an organization that
must have no subscription, a queue that must start empty.

**Failure mode without it:** every hour spent investigating a product defect that was a
fixture, and the credibility cost of reporting one.

**Cost.** One assertion per assumed property, and you must know what your fixtures actually
are.

---

# The grammar

R1 requires that something other than a model can evaluate an assertion. One rule makes
that achievable without giving up readable specifications:

> **Anything that decides a verdict lives inside a typed block. Prose is explicitly
> non-normative.**

Prose is why humans review these documents. It just cannot be load-bearing.

## Typed blocks

| Block | Contains | Failure produces |
|---|---|---|
| `requires` | Preconditions: fixtures, credentials, seeds | `BLOCKED`, naming the requirement |
| `setup` | Steps before the stimulus. May capture and may assert. | `FAIL`, reason `setup` |
| `control` | A positive control for an inverted-verdict scenario | `FAIL`, reason `control` |
| `request` | The stimulus — a request, an event publish, a command | — |
| `capture` | Values bound for later assertions | `FAIL` if unresolved (R3) |
| `expect response` | Assertions on the immediate result | `FAIL` |
| `expect side-effect` | Assertions on downstream observable state | `FAIL` |
| `deviation` | Declares that the asserted behavior contradicts a **cited** intent. Pairs `intent` with `observed`. | `KNOWN-DEVIATION` when the deviation persists; `FAIL` reason `deviation_drifted` when the actual matches neither ([`02`](02-verdicts.md)) |
| `teardown` | Cleanup. Mandatory for any scenario that mutates state. | reported, does not change the verdict |

`setup` and `control` exist because multi-step scenarios otherwise put verdict-deciding
steps in prose, which the rule above forbids. **A scenario whose setup or control fails did
not test the behavior** — but it is `FAIL`, not `INFRA`: the system *was* exercised and
something in it did not do what the scenario required. Adjudicate harness-versus-product
afterward, in the open, rather than pre-classifying it as environment noise.

Anything outside these blocks — headings, tables, paragraphs, notes — is documentation.

## Value expressions

| Form | Meaning |
|---|---|
| `mint(uuid)` · `mint(email)` · `mint(slug)` | Generate a value unique to this execution |
| `now()` · `now() + <n>s` | Current time at evaluation |
| `{{VAR}} <op> <literal>` | Arithmetic on a captured numeric (`+ - * /`) |
| `response.body.<path>` · `response.header[<name>]` | Read from the recorded response |
| `<probe>(<target>).<selector>` | Read from a probe |

## Temporal wrappers — three, and they are not interchangeable

This is the distinction that most often goes wrong. Choosing the wrong wrapper produces a
scenario that passes at t=0 against exactly the defect it was written for.

| Wrapper | Semantics | Passes when |
|---|---|---|
| `within <budget>` | **Appearance.** Poll until true; first true evaluation wins. | The condition becomes true at any point inside the budget |
| `hold <budget>` | **Invariant.** Sample throughout; must be true at every sample **and** at budget expiry. | The condition is true continuously for the full budget |
| `absent through <budget>` | **Inverted.** Sample throughout; any match fails. | No match occurs for the full budget |

**Any assertion of the form "still equals", "exactly N", or "unchanged" MUST use `hold`.**
Written with `within`, it is satisfied the instant it is first true — which for a
count-that-must-not-grow is usually before the system could have done the wrong thing.

```
✗   store(refunds).count where order_id == {{ORDER_ID}}  within cross_service  == 1
✓   store(refunds).count where order_id == {{ORDER_ID}}  hold cross_service    == 1
```

The first is satisfied at t=0 by the refund that already exists. The second requires the
count to still be 1 when the budget expires — which is the actual claim.

The rule is about **invariants**: subjects that already exist as the stimulus is sent. The
same shape used as a **receipt** — `count == 1` for something the stimulus creates, pinned to
a correlator minted in this scenario — is an appearance claim and correctly takes `within`.
[`09-gates.md`](09-gates.md) states the mechanical form of the distinction.

`hold` on its own does not prove the stimulus arrived. Pair it with evidence of receipt —
see [`12-false-greens.md`](12-false-greens.md) B6.

## Assertion vocabulary

Keep it small. A large vocabulary is one a runner cannot fully implement and an agent will
improvise around.

### On a response

| Form | Meaning |
|---|---|
| `status == <n>` | Status equals |
| `body.<path> == <literal>` / `!= <literal>` | Field equals / differs. Dot paths; numeric segments index arrays |
| `body.<path> is <type>` | Type check, value-agnostic |
| `body.<path> exists` / `absent` | Presence |
| `body.<path> contains <literal>` | Substring for strings, membership for arrays |
| `body.<path> matches /regex/` | Pattern |
| `body.<path> count == <n>` | Array length |
| `header.<Name> == <literal>` / `contains <literal>` | Response header |
| `response == {{VAR}} ignoring <fields>` | Byte-comparison against a captured response |

### On a probe result

| Form | Meaning |
|---|---|
| `<probe>(<t>).<selector> where <pred>` | Select evidence. `<pred>` may combine terms with `and` |
| `… exists` / `… absent` | The selection is non-empty / empty |
| `… count == <n>` / `>= <n>` | Cardinality of the selection |
| `… .<field> == <literal>` | A field on the selected evidence |
| `… moved` / `… moved by >= <n>` | A counter advanced. **Never** an exact delta ([`06`](06-side-effects.md)) |

### In teardown

| Form | Meaning |
|---|---|
| `request … / expect status == <n>` | Teardown through the product. **Preferred** — a free extra assertion |
| `store(<t>).delete(<key>)` | Direct delete. Escape hatch; `mutable` environments only |
| `store(<t>).restore(<key>, <field> = <value>)` | Direct restore. Escape hatch |
| `queue(<t>).purge where <pred>` | Remove test residue |

## Prefer exact equality

Where the value is deterministic, assert the literal. `is string` where the value is known
to be `"ACTIVE"` is an under-assertion — and under-assertion is the false green that
survives every other rule here: the correlator is right, the evidence is real, the
assertion evaluates true, and the check proves nearly nothing.

The same applies to collections. If the request posted two known items, assert
`body.data.items count == 2` and the values — not `is array`.

## Assert stable tokens, not human copy

Where a response carries both a stable machine-readable code and a human-readable message,
assert the code. Message text gets reworded, and a specification that fails on a copy edit
will be weakened until it fails on nothing.

This applies to **log and event evidence too**. Assert structured fields — `event`,
`error_code`, `reason` — not `.message == "…"`. A log message is corroboration; a
structured field is contract.

A changed stable code **should** fail. That is a breaking change, and the specification
catching it is the specification working.
