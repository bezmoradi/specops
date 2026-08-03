# 10 — Environments

> A specification that contains a hostname, a region, a table name, and a cluster command
> is bound to one deployment forever.

The altitude rule ([`04-spec-altitude.md`](04-spec-altitude.md)) keeps implementation
details out of specifications. This page keeps *infrastructure* details out — the same
discipline applied to the other axis, and the one people miss because infrastructure feels
like part of the test rather than part of the environment.

## The rule

**Specifications reference logical names. A binding file maps logical to physical, per
environment.**

```
specification says          binding resolves to
─────────────────────       ──────────────────────────────────────
host(api)                   https://api.staging.example.com
store(orders)               the orders table, with this environment's prefix
log(payments)               however logs are read here
event(billing)              this environment's billing topic
```

Change environment, change one file. The specifications never move.

## What belongs in the binding

Everything that varies by deployment, and nothing else:

| Category | Examples |
|---|---|
| **Endpoints** | Host per logical surface. Systems with more than one entry point need one per surface — routing them wrong is a false green. |
| **Data stores** | Logical name to physical name, including any environment prefix |
| **Messaging** | Logical topic and queue names to physical |
| **Evidence access** | How logs are read here; how metrics are read here; the replica selector |
| **Budgets** | Poll budgets, which legitimately differ between a laptop-adjacent environment and a loaded one |
| **Safety class** | Whether this environment permits mutation and escape hatches at all |
| **Fixture references** | Identifiers of the permanent fixtures that exist here |

```yaml
# environments/staging.yaml
name: staging
safety: mutable            # mutable | read-only | forbidden

hosts:
  api:  https://api.staging.example.com
  data: https://data.staging.example.com

stores:
  orders:   { prefix: "staging_", name: "orders" }
  customers:{ prefix: "staging_", name: "customers" }

events:
  billing:  { topic: "staging-billing", sink: "specops-sink-billing" }

evidence:
  logs:
    orders:   { selector: "app=orders-service",   aggregate_replicas: true }
    payments: { selector: "app=payments-service", aggregate_replicas: true }

budgets:
  single_hop: 15s
  cross_service: 30s
  index_propagation: 15s

fixtures:
  permanent_accounts: ["acct_0001", "acct_0002"]
```

## Safety class

**The environment declares whether it may be mutated. The runner enforces it.**

```yaml
safety: mutable      # normal verification environment
safety: read-only    # production: probes may observe, nothing may be written
safety: forbidden    # never target this
```

This matters more than it looks. Environment safety is usually enforced by convention —
a rule in a document, a habit, a guard in one script. Conventions fail under time
pressure, at the exact moment a mistake is most expensive.

Make it structural:

1. **The runner refuses to execute a mutating step against a non-`mutable` environment.**
   Not a warning. A refusal.
2. **Escape hatches require `mutable`.** No exceptions, no override flag that is convenient
   enough to become habitual.
3. **The static gate checks it** ([`09-gates.md`](09-gates.md)) — no specification may
   reference a physical production identifier directly, because doing so bypasses the
   binding and therefore the check.
4. **Every bundle records which environment it ran against**
   ([`08-evidence-bundles.md`](08-evidence-bundles.md)), so an accident is at least
   discoverable afterward.

The general principle: a safety property enforced by prose is enforced only while everyone
is paying attention. Move it into a place where violating it requires editing a file
someone will review.

## Verifying production

Some verification genuinely belongs in production — it is the only place with real traffic,
real data volumes, and real third-party integrations.

A `read-only` environment makes a restricted subset possible:

- **Permitted:** health and readiness checks, observing that a metric is live, asserting
  the shape of a public unauthenticated response, confirming that a scheduled job's output
  exists.
- **Not permitted:** anything that creates, mutates, or deletes. Any escape hatch. Anything
  needing a fixture, since provisioning one is a mutation.

Keep the production-eligible subset explicitly tagged rather than inferred. "It looked
read-only" is not a safety argument, and a scenario can become mutating through an edit to
a `requires` block that nobody thought of as risky.

## Multiple entry points

Systems that expose more than one entry point — an API host and a separate host for a
machine-facing surface, say — create a specific and nasty false green: a request sent to
the wrong host may be answered by a *different service* that happens to return something
plausible.

Three defenses:

1. **The header table declares the surface** ([`04-spec-altitude.md`](04-spec-altitude.md)).
2. **The binding maps surface to host**, so the specification never names a host.
3. **A preflight assertion at run start** proves the routing is what you think it is:
   request a path that only the intended service serves, and assert its distinctive
   response. Routing configuration drifts, and drift is invisible until something passes
   against the wrong service.

## Executor privilege

This method quietly requires handing an agent a **cross-system operator credential**: store
writes in a verification environment, and log and store *reads* — which routinely include
personal data — potentially in production. That is a larger grant than any conventional test
runner receives, and it is invisible unless stated.

Treat the executor as an operator, not as a test:

1. **A dedicated identity per environment.** Never a human's credential, never one shared
   with the application.
2. **Scoped to the bindings.** The executor's grants should be derivable from the
   environment file: exactly the stores, topics, and log streams the bindings name — nothing
   wider. A binding that names one store should not come with access to all of them.
3. **`read-only` environments get read-only credentials.** The `safety` class is enforced by
   the runner *and* by the credential, so a runner bug cannot become a production mutation.
4. **Reads are audited.** The executor's access appears in your normal audit trail, with a
   distinguishable identity, so "what did the test harness read last Tuesday" is answerable.
5. **Personal data is redacted at capture.** Bundles will otherwise contain whatever the
   logs contain ([`08-evidence-bundles.md`](08-evidence-bundles.md)).
6. **Credentials are short-lived and rotated**, on the same schedule as any other operator
   credential.

The uncomfortable framing is the accurate one: you are granting an autonomous process broad
read access to production data in order to verify behavior. That may well be worth it. It
should be a decision someone made, not a side effect of adopting a test method.

## Configuration is part of the environment

Verification transfers between environments only as far as **configuration** matches.

A suite green in staging with a feature flag on says nothing about production with the flag
off. The header-table gate ([`09-gates.md`](09-gates.md)) covers rate-limit tiers because
those are declared per specification; it covers nothing else.

Two mitigations, and the first is cheap:

- **Declare the configuration a suite assumes.** List, in the environment file, the flags
  and settings the corpus depends on, with their expected values. A run asserts them at
  start and fails loudly on divergence — the same shape as the routing preflight.
- **Diff configuration across environments before promoting.** Any key that differs is a
  scenario whose result does not transfer, and it should be named in the run ledger's "not
  run" section rather than assumed.

```yaml
assumed_config:
  orders.async_refunds: true
  orders.max_items: 999
  auth.session_ttl: 8h
```

## Per-environment budgets

Poll budgets belong in the binding, not in specifications. The same assertion may need 5
seconds against a lightly loaded environment and 30 against a busy one, and the *behavior*
being asserted is identical.

Specifications reference a named budget class:

```
event(billing).published where order_id == {{ORDER_ID}}   within cross_service
```

This also gives you a lever: if a whole environment is slow, you tune one file rather than
editing hundreds of specifications — which is what people do instead, badly, by adding
retries.
