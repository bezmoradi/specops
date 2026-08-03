# Examples

Three worked specifications for a fictional orders system. They are complete rather than
illustrative — you should be able to read one and see exactly what a real specification
looks like.

| Example | Teaches |
|---|---|
| [`guide.md`](guide.md) | **A filled-in execution contract.** The artifact adopters skip, and why two runs of the same suite otherwise mean different things. |
| [`rest-crud/orders-create.md`](rest-crud/orders-create.md) | The base case: request, response, persisted state, published event. Correlators, budgets, teardown, provenance tags — plus an intended non-event with a proxy control (OC3). |
| [`async-event/order-cancelled.md`](async-event/order-cancelled.md) | The stimulus is an event publish and the verdict is entirely the side effect. **FC2 is the worked example of `hold` versus `within`** — the invariant that passes at t=0 if you get it wrong. |
| [`negative-space/orders-tenancy.md`](negative-space/orders-tenancy.md) | Authorization, tenancy isolation, the existence oracle, seed-visibility controls, and a bracketed proxy control. The hardest specifications to get right. |
| [`bundle/orders-create-OC1.json`](bundle/orders-create-OC1.json) | **A real evidence bundle.** What you actually read in adoption Step 6, instead of the verdict. |

## The system these describe

A small orders service:

- `POST /v1/orders` creates an order, persists it, publishes `order.created` to the
  `billing` topic.
- `GET /v1/orders/{id}` reads one, scoped to the caller's tenant.
- A queue consumer handles `order.cancelled`: it releases reserved inventory, writes a
  refund record, and notifies the customer.
- Callers authenticate with a bearer token carrying a tenant identifier.
- Responses use a single envelope with a stable `code` field.

Nothing about the implementation is stated, because a specification never states it. Note
that you can read all three examples without ever learning what language the service is
written in — that is the altitude rule working
([`method/04-spec-altitude.md`](../method/04-spec-altitude.md)).

## Reading them

Each example uses the typed blocks from
[`templates/spec.md`](../templates/spec.md). Prose is non-normative — it explains, it never
decides a verdict.

Inline comments marked `←` point at the rule being demonstrated and link to the page that
states it. Those comments would not appear in a real specification.
