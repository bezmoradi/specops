---
id: fanout-order-cancelled
transport: queue
path: orders.cancellations
surface: queue
auth: n/a
tier: async_consumer
envelope: n/a
tags: [orders, fanout, cascade]
---

# Event: `order.cancelled`

**Conventions:** see [../guide.md](../guide.md).

The stimulus here is an **event publish**, not an HTTP request. There is no response to
assert — the verdict is **entirely** the side effect. The consumer must release reserved
inventory, write a refund record, and dispatch a customer notification.

Event shape:

```json
{
  "event_type": "order.cancelled",
  "event_id":   "<uuid, minted per execution>",
  "timestamp":  "<ISO-8601>",
  "data": { "order_id": "<uuid>", "tenant_id": "<uuid>", "reason": "customer_request" }
}
```

The suite publishes **directly to the queue**, bypassing the topic's subscription filter —
that removes one hop of non-determinism and isolates the event to this consumer, so no other
service is signalled by a test.

The consumer deduplicates on `event_id`, so **every scenario mints a fresh one**. Reusing
one means the stimulus is correctly ignored and the scenario observes nothing
([`method/12`](../../method/12-false-greens.md) F4).

`tier: async_consumer` assigns these scenarios a lane even though no HTTP rate limiter
applies; see [`../guide.md`](../guide.md) §9.

---

## FC1 — A cancellation releases inventory, refunds, and notifies

**Behavior:** publishing `order.cancelled` for a real order releases its reserved
inventory, writes a refund record, and dispatches a notification.
**Provenance:** @intent — ORD-31, "cancelling an order must release inventory within one
minute and refund in full"
**Tags:** @orders @fanout @cascade @side-effect @destructive

```requires
fixture: tenant_a_token
fixture: tenant_a_customer
```

```setup
# A per-scenario SKU. An earlier version reserved against a shared catalogue row, which
# made `reserved == {{BEFORE}} - 2` flake under any concurrent reservation and made
# teardown's restore write a stale value over other lanes' activity.
SKU      = mint(slug)
EVENT_ID = mint(uuid)

request POST {{surface(api)}}/v1/catalogue
        Authorization: Bearer {{TENANT_A_TOKEN}}
        { "sku": "{{SKU}}", "price_cents": 1000, "stock": 100 }
expect  status == 201

request POST {{surface(api)}}/v1/orders
        Authorization: Bearer {{TENANT_A_TOKEN}}
        { "customer_id": "{{TENANT_A_CUSTOMER}}",
          "items": [{ "sku": "{{SKU}}", "quantity": 2 }] }
expect  status == 201
capture ORDER_ID = response.body.data.id

# The reservation must be real before "released" means anything — otherwise the
# assertion below cannot distinguish released from never-reserved. method/12 A3.
expect  store(inventory).item({{SKU}})  within index_propagation
        .reserved == 2
```

```request
publish queue(orders.cancellations)
{
  "event_type": "order.cancelled",
  "event_id":   "{{EVENT_ID}}",
  "timestamp":  "{{now()}}",
  "data": {
    "order_id":  "{{ORDER_ID}}",
    "tenant_id": "{{TENANT_A}}",
    "reason":    "customer_request"
  }
}
```

```expect side-effect
# ← State is the authoritative signal: it proves the OUTCOME, not that a handler ran.

store(orders).item({{ORDER_ID}})                            within cross_service
  .status == "cancelled"

store(inventory).item({{SKU}})                              within cross_service
  .reserved == 0

store(refunds).item where order_id == {{ORDER_ID}}          within cross_service
  .status        == "issued"
  .amount_cents  == 2000

# ← Structured fields, not message copy. An earlier version asserted
#   `.message == "order cancellation processed"`, which is E2 inside the examples.
log(orders).line where event_id == {{EVENT_ID}}             within cross_service
  .event    == "order.cancellation.processed"
  .order_id == "{{ORDER_ID}}"

# ← Corroboration ONLY — a counter cannot carry a per-execution identifier. method/12 B4.
metric(orders.cancellations.processed) moved
```

```teardown
store(refunds).delete where order_id == {{ORDER_ID}}   # escape-hatch (mutable only)
store(orders).delete({{ORDER_ID}})                     # escape-hatch (mutable only)

request DELETE {{surface(api)}}/v1/catalogue/{{SKU}}
        Authorization: Bearer {{TENANT_A_TOKEN}}
expect  status == 204
```

**Notes**

Teardown deletes the order through an escape hatch because a cancelled order has no delete
endpoint — the product deliberately retains cancelled orders. This is the narrow case
escape hatches exist for: state no endpoint can reach. Note that the escape hatch is a
**write**; the `store(...)` reads in `expect side-effect` above are not escape hatches, they
are the top evidence tier ([`method/04`](../../method/04-spec-altitude.md)).

Because the SKU is per-scenario, no `restore` is needed — the catalogue row is deleted
outright.

---

## FC2 — Redelivery of the same event is idempotent

**Behavior:** the same `event_id` delivered twice produces exactly one refund.
**Provenance:** @intent — ORD-31 requires at-most-once refunding under at-least-once
delivery
**Tags:** @orders @fanout @regression @side-effect @destructive

```requires
fixture: tenant_a_token
fixture: tenant_a_customer
```

```setup
SKU      = mint(slug)
EVENT_ID = mint(uuid)

request POST {{surface(api)}}/v1/catalogue
        Authorization: Bearer {{TENANT_A_TOKEN}}
        { "sku": "{{SKU}}", "price_cents": 1000, "stock": 100 }
expect  status == 201

request POST {{surface(api)}}/v1/orders
        Authorization: Bearer {{TENANT_A_TOKEN}}
        { "customer_id": "{{TENANT_A_CUSTOMER}}",
          "items": [{ "sku": "{{SKU}}", "quantity": 2 }] }
expect  status == 201
capture ORDER_ID = response.body.data.id

publish queue(orders.cancellations)
        { "event_type": "order.cancelled", "event_id": "{{EVENT_ID}}",
          "timestamp": "{{now()}}",
          "data": { "order_id": "{{ORDER_ID}}", "tenant_id": "{{TENANT_A}}",
                    "reason": "customer_request" } }

expect  store(refunds).item where order_id == {{ORDER_ID}}  within cross_service
        exists
capture REFUND_ID = store(refunds).item where order_id == {{ORDER_ID}} .id
```

```request
publish queue(orders.cancellations)
# byte-identical to the setup payload, INCLUDING {{EVENT_ID}}
{
  "event_type": "order.cancelled",
  "event_id":   "{{EVENT_ID}}",
  "timestamp":  "{{now()}}",
  "data": {
    "order_id":  "{{ORDER_ID}}",
    "tenant_id": "{{TENANT_A}}",
    "reason":    "customer_request"
  }
}
```

```expect side-effect
# ← STEP 1: prove the duplicate was RECEIVED. Without this the invariant below is
#   satisfied by zero delivery — the consumer could be dead and the scenario green.
#   method/12 B6.
log(orders).line where event_id == {{EVENT_ID}} and event == "event.duplicate.ignored"
  within cross_service
  count == 1

# ← STEP 2: `hold`, NOT `within`. The count is already 1 the instant the duplicate is
#   published, so an appearance poll passes at t=0 — before the consumer could possibly
#   have double-refunded. `hold` requires the count to still be 1 at budget expiry,
#   which is the actual claim. This is the exact defect this scenario exists to catch.
store(refunds).count where order_id == {{ORDER_ID}}
  hold cross_service
  == 1

store(refunds).item where order_id == {{ORDER_ID}}
  .id == "{{REFUND_ID}}"
```

```teardown
store(refunds).delete where order_id == {{ORDER_ID}}   # escape-hatch (mutable only)
store(orders).delete({{ORDER_ID}})                     # escape-hatch (mutable only)

request DELETE {{surface(api)}}/v1/catalogue/{{SKU}}
        Authorization: Bearer {{TENANT_A_TOKEN}}
expect  status == 204
```

**Notes**

This scenario is the worked example for
[`method/12`](../../method/12-false-greens.md) B6. An earlier version asserted
`count == 1 within cross_service` with no receipt evidence, and was satisfied at t=0 by the
refund the setup had already waited for. It passed against every double-refunding system
with any processing delay — the precise defect it was written to catch.

---

## FC3 — A malformed payload is not silently dropped

**Behavior:** an event missing `order_id` cannot be processed, and the consumer surfaces
it rather than discarding it.
**Provenance:** @contract
**Tags:** @orders @fanout @regression @side-effect

```setup
EVENT_ID = mint(uuid)
```

```request
publish queue(orders.cancellations)
{
  "event_type": "order.cancelled",
  "event_id":   "{{EVENT_ID}}",
  "timestamp":  "{{now()}}",
  "data": { "tenant_id": "{{TENANT_A}}", "reason": "customer_request" }
}
```

```expect side-effect
log(orders).line where event_id == {{EVENT_ID}}      within cross_service
  .level      == "error"
  .event      == "event.rejected"
  .error_code == "MISSING_ORDER_ID"

# The message must reach the dead-letter queue rather than being discarded — that
# distinction is the whole point of this scenario.
queue(orders.cancellations.dlq).message where event_id == {{EVENT_ID}}
  within cross_service
  exists
```

```teardown
queue(orders.cancellations.dlq).purge where event_id == {{EVENT_ID}}
```

**Notes**

**Expected output, not a defect:** while this scenario runs, the consumer logs the same
event failing several times as the queue redelivers it up to its retry limit, and the
dead-letter depth ticks up. That is the scenario working. Do not report the repeated
processing or the dead-letter residue as a failure.

Assertions are on `event` and `error_code` rather than message text — a reworded log message
must not fail a specification ([`method/03`](../../method/03-evidence-rules.md)).
