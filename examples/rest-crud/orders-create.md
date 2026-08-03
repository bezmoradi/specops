---
id: orders-create
method: POST
path: /v1/orders
surface: api
auth: bearer
tier: mutation
envelope: default
tags: [orders, regression]
---

# POST /v1/orders

**Conventions:** see [../guide.md](../guide.md).

Creates an order for the authenticated caller's tenant. The order is persisted and an
`order.created` event is published to the `billing` topic.

Request body:

- `customer_id` (string, required) — must belong to the caller's tenant
- `items[]` (array, required, min 1)
  - `sku` (string, required)
  - `quantity` (integer, required, 1–999)
- `note` (string, optional, max 500)

A customer may hold at most one order in `pending` state. A second attempt is rejected.

---

## OC1 — A valid order is created and published

**Behavior:** an authenticated caller creates an order; it is persisted as `pending` and
an `order.created` event is published.
**Provenance:** @intent — ORD-14, "creating an order reserves it for billing"
**Tags:** @orders @smoke @side-effect @destructive

```requires
fixture: tenant_a_token
fixture: tenant_a_customer
```

```setup
SKU = mint(slug)                       # per-scenario SKU: the catalogue row is shared state
request POST {{surface(api)}}/v1/catalogue
        Authorization: Bearer {{TENANT_A_TOKEN}}
        { "sku": "{{SKU}}", "price_cents": 1000, "stock": 100 }
expect  status == 201
```

```request
POST {{surface(api)}}/v1/orders
Authorization: Bearer {{TENANT_A_TOKEN}}
Content-Type: application/json

{
  "customer_id": "{{TENANT_A_CUSTOMER}}",
  "items": [{ "sku": "{{SKU}}", "quantity": 2 }]
}
```

```expect response
status == 201                                    @intent
body.code == "ORDER_CREATED"                     @contract
body.data.id exists                              @contract
body.data.status == "pending"                    @intent
body.data.customer_id == "{{TENANT_A_CUSTOMER}}" @contract
body.data.items count == 1                       @contract
body.data.items.0.sku == "{{SKU}}"               @contract
body.data.items.0.quantity == 2                  @contract
body.data.total_cents == 2000                    @contract
```

<!--
← method/03 — exact literals where the value is deterministic. An earlier version
asserted `items is array`, which is true of any array and was E1 sitting next to a
comment about exact literals. Because the scenario mints its own SKU at a known
price, total_cents is deterministic too and is asserted exactly.
-->

```capture
ORDER_ID = response.body.data.id
TRACE_ID = response.header["X-Trace-Id"]
```

```expect side-effect
# ← method/06 — response and side effect are INDEPENDENT verdicts.

store(orders).item({{ORDER_ID}})                          within index_propagation
  .status      == "pending"
  .customer_id == "{{TENANT_A_CUSTOMER}}"
  .tenant_id   == "{{TENANT_A}}"

# ← RECEIPT. Pinned on ORDER_ID, which did not exist before this request. A predicate
#   matching only on event_type keeps matching a PREVIOUS execution's message for as long
#   as the topic retains it. method/12 B2.
#
#   `count == 1`, not `exists`: creating an order must publish ONE event, and `exists` is
#   true of one message and equally true of two. method/12 H1.
event(billing).published where order_id == {{ORDER_ID}}   within cross_service
  count == 1
  .event_type  == "order.created"
  .tenant_id   == "{{TENANT_A}}"
  .total_cents == 2000

# ← BRACKET, and it is not optional. The receipt above uses `within` — correctly, since
#   the subject did not exist before the stimulus — but `within` is first-true-wins: it
#   is satisfied the instant the count reaches 1 and stops looking. A duplicate published
#   a moment later, or on a retry, or on a redelivery, lands after it stopped and passes
#   forever.
#
#   The budget is `single_hop`, not `cross_service`: it should cover the RETRY interval,
#   which is the window a duplicate has to arrive in. method/06, method/12 H1.
event(billing).published where order_id == {{ORDER_ID}}   hold single_hop
  count == 1
```

```teardown
request DELETE {{surface(api)}}/v1/orders/{{ORDER_ID}}
        Authorization: Bearer {{TENANT_A_TOKEN}}
expect  status == 204

request DELETE {{surface(api)}}/v1/catalogue/{{SKU}}
        Authorization: Bearer {{TENANT_A_TOKEN}}
expect  status == 204
```

<!--
← method/07 — teardown through the product's own endpoint is a free extra assertion:
a broken delete surfaces as a finding instead of hiding.
-->

---

## OC2 — A second pending order for the same customer is rejected

**Behavior:** a customer may hold at most one pending order; the second attempt is
rejected and nothing is published.
**Provenance:** @intent — ORD-14
**Tags:** @orders @regression @side-effect @destructive

```requires
fixture: tenant_a_token
fixture: tenant_a_customer
```

```setup
SKU = mint(slug)
request POST {{surface(api)}}/v1/catalogue
        Authorization: Bearer {{TENANT_A_TOKEN}}
        { "sku": "{{SKU}}", "price_cents": 1000, "stock": 100 }
expect  status == 201

request POST {{surface(api)}}/v1/orders
        Authorization: Bearer {{TENANT_A_TOKEN}}
        { "customer_id": "{{TENANT_A_CUSTOMER}}",
          "items": [{ "sku": "{{SKU}}", "quantity": 1 }] }
expect  status == 201
capture FIRST_ORDER_ID = response.body.data.id
```

<!--
← The setup is a typed block, not prose. Its assertions decide the verdict — if the
first order was not created, this scenario tests nothing — and method/03 requires
anything that decides a verdict to live in a typed block.
-->

```request
POST {{surface(api)}}/v1/orders
Authorization: Bearer {{TENANT_A_TOKEN}}
Content-Type: application/json

{
  "customer_id": "{{TENANT_A_CUSTOMER}}",
  "items": [{ "sku": "{{SKU}}", "quantity": 1 }]
}
```

```expect response
status == 409                                        @intent
body.code == "ORDER_ALREADY_PENDING"                 @contract
body.data.existing_order_id == "{{FIRST_ORDER_ID}}"  @contract
```

```expect side-effect
# ← `hold`, not `within`. The count is already 1 when the request is sent, so an
#   appearance poll would be satisfied at t=0 and a system that created a second
#   order and failed afterwards would pass. method/12 B6.
store(orders).count where customer_id == {{TENANT_A_CUSTOMER}} and status == "pending"
  hold cross_service
  == 1
```

```teardown
request DELETE {{surface(api)}}/v1/orders/{{FIRST_ORDER_ID}}
        Authorization: Bearer {{TENANT_A_TOKEN}}
expect  status == 204

request DELETE {{surface(api)}}/v1/catalogue/{{SKU}}
        Authorization: Bearer {{TENANT_A_TOKEN}}
expect  status == 204
```

---

## OC3 — An unauthenticated request is rejected and publishes nothing

**Behavior:** a request with no credential is rejected before any work occurs, and no
event is published.
**Provenance:** @contract
**Tags:** @orders @security @no-auth @side-effect

```requires
fixture: tenant_a_token
fixture: tenant_a_customer
```

```setup
MARKER      = mint(uuid)      # a value nothing else in the corpus will ever produce
CONTROL_SKU = mint(slug)
request POST {{surface(api)}}/v1/catalogue
        Authorization: Bearer {{TENANT_A_TOKEN}}
        { "sku": "{{CONTROL_SKU}}", "price_cents": 1000, "stock": 100 }
expect  status == 201
```

```control
# ← A PROXY control (method/06): the target event for this scenario can only be
#   produced by the very bug under test, so no legitimate action produces a matching
#   event. This control proves the topic → sink path is live end-to-end.
#   Residual gap: it does not prove routing for a payload carrying {{MARKER}}.
request POST {{surface(api)}}/v1/orders
        Authorization: Bearer {{TENANT_A_TOKEN}}
        { "customer_id": "{{TENANT_A_CUSTOMER}}",
          "items": [{ "sku": "{{CONTROL_SKU}}", "quantity": 1 }] }
expect  status == 201
capture CONTROL_ORDER_ID = response.body.data.id
expect  event(billing).published where order_id == {{CONTROL_ORDER_ID}}
        within cross_service
        exists
```

```request
POST {{surface(api)}}/v1/orders
Content-Type: application/json

{
  "customer_id": "{{MARKER}}",
  "note": "{{MARKER}}",
  "items": [{ "sku": "{{MARKER}}", "quantity": 1 }]
}
```

```expect response
status == 401
body.code == "UNAUTHORIZED"
```

```expect side-effect
# ← BROAD predicate over a unique marker. method/03 R3: breadth inverts for absence.
#   An earlier version pinned `customer_id == "any"` — narrow, and on a shared
#   subject: a pre-auth publish emitting a null customer_id would have missed the
#   predicate entirely and the scenario would have gone green.
event(billing).published where any_field contains {{MARKER}}
  absent through cross_service
```

```control
# ← Bracket. A control only at the front proves the pipeline was live when the window
#   opened, not that it stayed live through the absence poll. method/06.
expect  event(billing).published where order_id == {{CONTROL_ORDER_ID}}
        exists
```

```teardown
request DELETE {{surface(api)}}/v1/orders/{{CONTROL_ORDER_ID}}
        Authorization: Bearer {{TENANT_A_TOKEN}}
expect  status == 204

request DELETE {{surface(api)}}/v1/catalogue/{{CONTROL_SKU}}
        Authorization: Bearer {{TENANT_A_TOKEN}}
expect  status == 204
```

---

## OC4 — Quantity above the documented maximum is rejected

**Behavior:** an item quantity above 999 is rejected as a validation failure.
**Provenance:** @intent — ORD-14 specifies a per-item maximum of 999
**Tags:** @orders @regression

```requires
fixture: tenant_a_token
fixture: tenant_a_customer
```

```request
POST {{surface(api)}}/v1/orders
Authorization: Bearer {{TENANT_A_TOKEN}}
Content-Type: application/json

{
  "customer_id": "{{TENANT_A_CUSTOMER}}",
  "items": [{ "sku": "SKU-CATALOGUE-STATIC", "quantity": 1000 }]
}
```

```expect response
status == 400                                   @intent
body.code == "VALIDATION_FAILED"                @contract
body.data.errors.0.field == "items.0.quantity"  @contract
```

**Notes**

The boundary is asserted from the requirement, not from the implementation's constant. This
is what provenance tagging exists for: if the code had been written with a maximum of 1000,
this scenario would fail — and it should, because the requirement says 999. See
[`method/05-provenance.md`](../../method/05-provenance.md).

This scenario is non-destructive (the request is rejected before anything is created), so it
uses the shared static catalogue row rather than minting one.
