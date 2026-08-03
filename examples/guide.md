# Execution contract — Orders (example)

A **filled-in** [`templates/guide.md`](../templates/guide.md) for the fictional orders
system the examples describe. This is the artifact people most often skip, and skipping it
is why two runs of the same suite can mean different things.

Specifications state **what** should be true. This states **what counts as having observed
it here.**

---

## 0 — What is different about this system

1. **Two entry points.** `surface(api)` serves `/v1/*` for callers. `surface(queue)` is the
   message transport — the suite publishes to it directly. A specification that sends an
   HTTP request to the wrong surface gets a plausible answer from the wrong component; the
   routing preflight in `environments/*.yaml` exists to catch that.
2. **The orders service runs two replicas.** Evidence for one execution lands on exactly
   one of them. Every log read aggregates.
3. **The queue consumer deduplicates on `event_id`.** Every published event must carry a
   freshly minted one, or the stimulus is silently a no-op.
4. **Cancelled orders are retained by design** — there is no delete endpoint for them.
   Teardown for cancellation scenarios must use an escape hatch.
5. **Refunds are issued asynchronously**, up to ~20s after cancellation. Every refund
   assertion uses the `cross_service` budget.

---

## 1 — Verdicts

Per [`method/02-verdicts.md`](../method/02-verdicts.md): `PASS` · `FAIL` · `BLOCKED` ·
`N/A` · `INFRA` · `INDETERMINATE`.

Local note: a scenario whose positive control fails to fire is **`FAIL`, reason `control`**
— never `INFRA`. The system was exercised and did not do what the scenario required
(tenant B cancelling its own order should publish). Filing that as environment noise takes a
product regression and puts it where nothing blocks on it. Adjudicate harness-versus-product
afterward, in the open.

---

## 2 — The rules, as local commitments

1. The agent never votes. Verdicts are computed from recorded evidence.
2. Fail closed. Missing, empty, or errored evidence is `FAIL`.
3. No empty captures, no weaker correlator.
4. Bounded poll, never re-run-until-green.
5. Altitude: no internal symbol, file, or line number in any specification.
6. Provenance: every assertion is tagged `@intent` or `@contract`.

---

## 3 — Correlators

| Situation | Correlator | Where it appears |
|---|---|---|
| Request-initiated side effect | `X-Trace-Id`, propagated to the topic and the consumer | Response header; `trace_id` on every log line |
| Suite-published event | the minted `event_id` | Echoed on consumer log lines |
| Newly created order | the returned `id` | `order_id` on events, refund records, and log lines |
| Per-scenario customer | `qa+<unix>@example.test` | `customer_id` on orders and events |

**Known gap.** The service never logs `customer_email` — it is treated as personal data.
Searching logs by email returns nothing, permanently, which fail-closed correctly turns
into a red that *looks* like a product failure. Bridge through `customer_id` instead.

---

## 4 — Evidence sources, ranked

| Source | Logical name | How it is read | Authoritative? |
|---|---|---|---|
| Order / refund / inventory state | `store(orders)`, `store(refunds)`, `store(inventory)` | Direct read. **This is evidence, not an escape hatch** — reads are sanctioned in any environment; only *writes* are restricted | **yes** |
| Application logs | `log(orders)` | Aggregated across both replicas by selector | **yes**, when correlated |
| Published events | `event(billing)` | The suite's private subscriber `specops-sink-billing` | **yes**, when pinned on a unique field |
| Dead-letter queue | `queue(orders.cancellations.dlq)` | Direct read | **yes** |
| Counters | `metric(orders.*)` | Scrape endpoint | **corroboration only** |

**Private sink.** `specops-sink-billing` is subscribed alongside the real billing consumer.
Fan-out gives it its own copy, so the suite reads the event **as published** without racing
the real consumer. It **peeks** rather than consumes, so parallel lanes can share it —
which means a broad predicate will keep re-matching an old message. Always pin a
per-execution-unique field.

**Boundary.** A sink capture proves *publish*, not *delivery*. Where the point is that the
consumer acted, assert the consumer's observable outcome too.

---

## 5 — Poll budgets

| Class | Covers | staging |
|---|---|---|
| `index_propagation` | Secondary index consistency | 15s |
| `single_hop` | One asynchronous delivery | 15s |
| `cross_service` | Publish → transport → consumer → state | 30s |

Values live in `environments/<env>.yaml`. Specifications reference the **class name** only.

---

## 6 — Authentication bootstrap

```
POST {{surface(api)}}/v1/auth/token
  { "client_id": "...", "client_secret": "..." }
  → 200, data.access_token → {{TENANT_A_TOKEN}}   (1h)
```

Fully automated. Tokens are refreshed transparently mid-run. Tenant B's token is minted the
same way into `{{TENANT_B_TOKEN}}`.

**Tenant B's token IS used for positive reads, deliberately.** An earlier version of this
guide said it never was — tenant B existed "only as a cross-tenant target." That rule made
the seed-visibility control impossible, and without that control the tenancy scenarios pass
against a system with no tenancy checks at all
([`method/12`](../method/12-false-greens.md) A4). Tenant B reads its own resources to
prove a seed is real to the product, and to prove a foreign resource survived a refused
mutation. It is never used as the *subject* of an isolation assertion.

---

## 7 — Fixtures

| Fixture | Category | Provisioned by | Torn down by |
|---|---|---|---|
| `tenant_a`, `tenant_b` | permanent | one-time setup | never |
| `SKU-CATALOGUE-STATIC` | permanent | one-time setup | never — read-only, used only by non-destructive scenarios |
| `tenant_a_token`, `tenant_b_token` | run-scoped | bootstrap | run end |
| `tenant_a_customer`, `tenant_b_customer` | run-scoped | provisioning | run end |
| minted SKUs, orders, refunds, foreign orders, control orders | scenario-scoped | the scenario's `setup` | the scenario's `teardown` |

**Every scenario that reserves inventory mints its own SKU.** The reservation counter is
shared mutable state: an arithmetic assertion against a shared row flakes under any
concurrent reservation, and a `restore` in teardown writes a stale value over another
lane's activity. Only scenarios whose request is rejected before anything is reserved may
use `SKU-CATALOGUE-STATIC`.

**Lifecycle:** clean slate. Everything except the two permanent tenants is created at run
start and destroyed at run end, because several scenarios assert freshness (an order's
initial `pending` status, an empty order list) and those are only honest against
freshly-provisioned fixtures.

**Steady state:** after teardown, exactly two tenants and zero orders. Teardown re-lists
and fails if that does not hold.

---

## 8 — Escape hatches

**An escape hatch is a WRITE.** Store *reads* are the top evidence tier in §4 and are not
escape hatches. Writes are permitted only in `requires` and `teardown`, only where
`safety: mutable`, and never as evidence of product behavior.

| Hatch (write) | Why no endpoint reaches this state |
|---|---|
| Seed an order owned by tenant B | Creating one through the API requires acting as tenant B inside a scenario whose caller must be tenant A |
| Seed an expired token | No endpoint mints one; waiting out a real expiry makes the scenario unrunnable. **Requires signing-key access — see the privilege note in `orders-tenancy.md` OT4** |
| Delete a cancelled order | Cancelled orders are deliberately retained; no delete endpoint exists |
| Delete a refund record | Same — refunds are immutable through the API by design |

Every seeded subject also needs a **seed-visibility control** before anything asserts it is
unreachable: a raw write can be invisible to the application, in which case the product
returns "not found" for everyone and the scenario passes for the wrong reason
([`method/12`](../method/12-false-greens.md) A4).

An earlier version of this table listed "restore an inventory count." That is gone: scenarios
now mint a per-scenario SKU and delete it outright, so nothing writes over shared counters.

---

## 9 — Parallelism

**Partition:** one lane per `tier` — `read`, `mutation`, `auth`, and `async_consumer`.
Derived from each specification's `tier` field, not maintained by hand.

`async_consumer` exists because event-driven specifications have no HTTP rate limiter but
still need a lane assignment. It is bounded by consumer throughput rather than a request
budget, and it runs concurrently with the HTTP lanes.

**Serial / quarantined:**

- `rate-limit.md` — floods the `read` tier; runs alone, last, with a 60s cooldown.

**Ordered chains:**

- `orders-tenancy.md` **OT1 → OT2**. OT2 consumes OT1's captured response to assert the two
  are indistinguishable. Same worker, in order, never interleaved.

OT3 is **not** serial. An earlier version marked it so on the grounds that its control
drained from a shared topic — but OT3 uses a *proxy* control, which is never drained, so
there was nothing to protect. Marking it serial cost wall-clock for no reason.

---

## 10 — Running

**There is no SpecOps binary.** These are the four steps every adopting repo needs; what
fills each slot is yours to choose. Today, most teams use a shell script plus an agent.

```bash
make specs-lint       # 1. STATIC GATE — your implementation of the method/09 checks.
                      #    No system, no model, seconds. Runs on every pull request.
                      #    Ours: a Python script over the corpus + the OpenAPI document.

make specs-provision  # 2. PROVISION — fixtures for the selected scenarios.
                      #    Ours: shell + the product's own API, ~90s.

make specs-run        # 3. RUN — hand the agent prompts/execute-specs.md, the selected
                      #    specifications, and the environment name. Emits evidence
                      #    bundles to runs/. Replay of cached plans where they exist.

make specs-teardown   # 4. TEARDOWN — destroy fixtures, then RE-LIST and assert the
                      #    steady state below. A teardown that half-completes silently
                      #    is how a clean-slate policy starts drifting.
```

If you build a runner, the conformance kit ([`method/09`](../method/09-gates.md)) defines
what it must do with each assertion form.

---

## 11 — Local false-green traps

Specific to this system. General catalogue:
[`method/12-false-greens.md`](../method/12-false-greens.md).

### The tenancy seed is load-bearing — twice over

`orders-tenancy.md` OT1 depends on two things that look deletable:

1. **The seed itself.** Without a real tenant-B order, the request returns 404 whether or
   not isolation is enforced, and the scenario passes against a system with no tenancy
   checks at all.
2. **The seed-visibility control.** The seed is a raw store write. If it did not populate
   whatever the read path needs, the product returns 404 for *everyone* — including tenant
   B — and the scenario passes again, for a different reason. Tenant B reading its own
   order and getting `200` is what rules that out.

Both keep passing when removed. That is exactly why someone will remove them.

### The refund invariant needs `hold`, not `within`

`order-cancelled.md` FC2 asserts `count == 1` with **`hold`**. With `within` it is satisfied
at t=0 by the refund the setup already waited for, and every double-refunding system with
any processing delay passes. It also asserts a dedup-hit log line first, because otherwise
zero-delivery of the duplicate satisfies the invariant trivially.

### `OT1` and `OT2` are only meaningful as a pair

The anti-enumeration property is that their responses **match**, so OT2 compares against
OT1's whole captured response rather than re-asserting three fields. Assert them
independently and both stay green while an extra field or a divergent header reintroduces
the oracle.

### The sink peeks, so broad predicates match old messages — except for absence

`specops-sink-billing` does not drain non-matching messages. For a **presence** assertion, a
predicate matching only `event_type == "order.created"` keeps matching the oldest lingering
message for as long as the topic retains it — including on a run where nothing was
published.

For an **absence** assertion the rule inverts: pin a per-execution-unique *subject* and match
**broadly** over it, because a narrow field pin misses a bug that publishes the same id under
a different field name. `OC3` and `OT3` both do this.

### A failed control is a product signal, not environment noise

Classifying it `INFRA` earns a sanctioned retry and files a product regression where nothing
blocks on it. `FAIL`, reason `control`, adjudicated afterward.
