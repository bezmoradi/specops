# 00 — SpecOps in one page

The whole method, quotable. Everything after this page is elaboration; read it when you hit
the situation it describes.

---

## The problem

An agent writes the implementation. An agent writes the tests. Both can encode the **same
misunderstanding** of the requirement, and then agree with each other perfectly. A green
suite says nothing.

And when an agent both *runs* a check and *decides whether it passed*, it rationalizes.
Shown a `201 Created` and no trace of the downstream email, it concludes the email probably
got sent. That is not an occasional lapse — it is what a system optimized to be helpful does
with an ambiguous result.

## Two separations

```
specification ──► implementation      an agent writes code from it
              ──► verification        a different agent proves the deployment matches it
```

The specification is the only artifact both sides share, and neither may edit it to make its
own work pass.

```
the agent   ACQUIRES evidence      non-deterministic — and that is the whole value
the rules   JUDGE the evidence     deterministic — and that is non-negotiable
```

The agent keeps the job it is good at: working out *how* to observe something across an API,
a queue, a log, a store, a metric. It loses the job it is bad at: deciding whether what it
found is good enough.

## Six rules

1. **The agent never votes.** A verdict is computed from recorded evidence against a declared
   assertion. Scoped precisely: no reasoning may move a scenario *toward* green. It is not a
   claim that the pipeline cannot be fooled.
2. **Fail closed.** Missing, empty, or errored evidence is `FAIL`. A `2xx` never excuses an
   absent side effect.
3. **No weaker correlator.** An empty correlator fails the scenario; it does not license a
   broader match that would alias someone else's evidence into your green.
4. **Bounded poll, never re-run-until-green.** Budgets absorb propagation delay. Re-running
   until it passes is how a real regression walks through a fail-closed gate.
5. **Altitude.** A specification states observable behavior and never names a class, a file,
   or a line. A behavior-preserving refactor must leave every specification valid.
6. **Provenance.** An assertion derived from the implementation is a regression check. One
   derived from a requirement is an oracle. Tag which, and keep the tag current.

## Seven verdicts, all computed

| | |
|---|---|
| `PASS` | Every assertion evaluated, every one true |
| `FAIL` | An assertion false, evidence absent within budget, or a setup/control failure |
| `KNOWN-DEVIATION` | Every assertion held, and what held contradicts a **cited** intent |
| `BLOCKED` | A declared precondition was unavailable — computed from `requires` |
| `N/A` | The specification declares it documented-not-driven |
| `INFRA` | A recorded probe error. Retries are legitimate, capped, and counted |
| `INDETERMINATE` | One of four enumerated acquisition failures |

Verdicts may be weakened, never strengthened. `KNOWN-DEVIATION` exists so that a *known*
defect need not be either a permanent red (which trains people to ignore red) or a silent
green (which the exit code reports as fine). It is counted separately and never folded into
`PASS`.

## Three temporal wrappers, and the one that goes wrong

| | Semantics |
|---|---|
| `within <budget>` | **Appearance** — poll until true; first true wins |
| `hold <budget>` | **Invariant** — true at every sample and at expiry |
| `absent through <budget>` | **Inverted** — any match fails |

Anything shaped *"still equals"*, *"exactly N"*, or *"unchanged"* uses **`hold`**. With
`within` it is satisfied the instant it is first true — which for a count that must not grow
is before the system could have done the wrong thing.

## A scenario

```markdown
## OC1 — A valid order is created and published
**Provenance:** @intent — ORD-14

​```requires
fixture: tenant_a_token
​```

​```request
POST {{surface(api)}}/v1/orders
Authorization: Bearer {{TENANT_A_TOKEN}}
{ "customer_id": "{{CUSTOMER}}", "items": [{ "sku": "{{SKU}}", "quantity": 2 }] }
​```

​```expect response
status == 201                          @intent
body.code == "ORDER_CREATED"           @contract
body.data.status == "pending"          @intent
​```

​```capture
ORDER_ID = response.body.data.id
​```

​```expect side-effect
store(orders).item({{ORDER_ID}})                        within index_propagation
  .status == "pending"
event(billing).published where order_id == {{ORDER_ID}}  within cross_service
  .event_type == "order.created"
​```

​```teardown
request DELETE {{surface(api)}}/v1/orders/{{ORDER_ID}}
expect  status == 204
​```
```

Prose explains and is **non-normative**. Anything that decides a verdict lives in a typed
block. That single line is what makes the format both readable by humans and evaluatable by
something other than a model.

## What it cannot do

Races and concurrency. Performance. Input-space coverage. Algorithmic correctness. Anything
with no observable consequence. Say this out loud when introducing it — a method that claims
completeness gets trusted with things it cannot do.

What it is unusually good at: **cross-system side effects** and **negative space**
(authorization, tenancy, existence oracles, events that must not fire). Which is where
production incidents actually live.

## The threat model

SpecOps defends against a **fallible** agent — one that rationalizes, drifts, or gets
sloppy. It does not defend against a **deceptive** one: a fabricated evidence bundle
recomputes perfectly, because recomputation checks internal consistency, not truth.

The partial defenses are machine-validating probe targets against environment bindings, plan
replay, and human bundle audit. Know their limits rather than over-reading rule 1.

## Where to go next

| | |
|---|---|
| **[`12-false-greens.md`](12-false-greens.md)** | **The catalogue — 43 ways a green suite lies to you.** If you read one more page, read this one. |
| [`01`](01-overview.md)–[`04`](04-spec-altitude.md) | The core argument: classification, verdicts, rules, altitude |
| [`05`](05-provenance.md)–[`07`](07-isolation.md) | Oracle versus regression net; evidence selection; running in parallel |
| [`08`](08-evidence-bundles.md)–[`11`](11-economics.md) | Operationalizing: auditability, gates, portability, cost |
| [`13-rule-audit.md`](13-rule-audit.md) | Every rule classified by what actually enforces it — mechanical, checkable-with-work, or judgment |
| [`../adoption/getting-started.md`](../adoption/getting-started.md) | Day one to first trustworthy green |
