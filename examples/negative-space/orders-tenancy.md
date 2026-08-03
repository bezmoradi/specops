---
id: orders-tenancy
method: GET
path: /v1/orders/{id}
surface: api
auth: bearer
tier: read
envelope: default
tags: [orders, security]
---

# Tenancy isolation — `/v1/orders/{id}`

**Conventions:** see [../guide.md](../guide.md).

The negative space: what the system must **refuse** to do, and what it must **not** reveal.
These are the hardest specifications to write correctly, because the natural way to write
them produces scenarios that cannot fail.

Two properties under test:

1. **Isolation** — a caller cannot read, mutate, or destroy another tenant's order.
2. **Anti-enumeration** — the response for "exists but belongs to someone else" is
   indistinguishable from "does not exist."

---

## OT1 — Another tenant's order is not readable, and is not revealed to exist

**Behavior:** a caller requesting an order belonging to a different tenant receives the
same response as for an order that does not exist.
**Provenance:** @intent — SEC-7, "a tenant may not determine whether another tenant's
resource exists"
**Tags:** @orders @security @escape-hatch @destructive

```requires
fixture: tenant_a_token
fixture: tenant_b_token
seed: order owned by tenant_b     # escape-hatch WRITE (mutable environments only)
```

```setup
# ── SEED-VISIBILITY CONTROL — load-bearing ────────────────────────────────────────
# The seed is written directly to the store. If the raw write did not populate
# everything the application's read path needs — a tenant-scoped index, a derived
# field, a cache — the product returns 404 for EVERYONE, the assertions below pass,
# and tenancy enforcement is never exercised. method/12 A4.
#
# So: prove the PRODUCT can see it, using the legitimate owner's credential, before
# asserting that anyone else cannot.
request GET {{surface(api)}}/v1/orders/{{FOREIGN_ORDER_ID}}
        Authorization: Bearer {{TENANT_B_TOKEN}}
expect  status == 200
expect  body.data.id == "{{FOREIGN_ORDER_ID}}"
expect  body.data.status == "pending"
```

<!--
← The other half of the load-bearing seed, from method/12 A3: without a real foreign
order at all, the request below returns 404 whether or not isolation is enforced,
so the scenario would pass against a system with NO tenancy checks.

Anyone "simplifying" this specification will be tempted to delete either the seed or
the visibility control, because the assertions keep passing without them. They keep
passing because the scenario has stopped testing anything.
-->

```request
GET {{surface(api)}}/v1/orders/{{FOREIGN_ORDER_ID}}
Authorization: Bearer {{TENANT_A_TOKEN}}
```

```expect response
status == 404                    @intent
body.code == "ORDER_NOT_FOUND"   @contract
body.data absent                 @intent
```

```capture
FORBIDDEN_RESPONSE = response
```

```expect side-effect
# ← Survival, via a PRODUCT read rather than a store read. A store read can be served
#   stale and show a row that has already been destroyed (method/12 B7); the product's
#   own read-your-write guarantee is stronger, and the altitude is better.
#   A destructive scoping bug — one that deleted the order while still returning 404 —
#   satisfies the response assertions above and is caught only here.
request GET {{surface(api)}}/v1/orders/{{FOREIGN_ORDER_ID}}
        Authorization: Bearer {{TENANT_B_TOKEN}}
expect  status == 200
        body.data.status == "pending"
        body.data.tenant_id == "{{TENANT_B}}"
```

```teardown
store(orders).delete({{FOREIGN_ORDER_ID}})    # escape-hatch (mutable only)
```

---

## OT2 — The absent-resource response is identical to the forbidden one

**Behavior:** requesting an id that does not exist returns exactly what OT1 returned, so
the two cases are indistinguishable to a caller.
**Provenance:** @intent — SEC-7
**Tags:** @orders @security @serial

```requires
fixture: tenant_a_token
capture: FORBIDDEN_RESPONSE from OT1
```

```request
GET {{surface(api)}}/v1/orders/00000000-0000-4000-8000-000000000000
Authorization: Bearer {{TENANT_A_TOKEN}}
```

```expect response
status == 404
body.code == "ORDER_NOT_FOUND"
body.data absent

# ← The property is that the two responses MATCH. Asserting three fields in each
#   scenario separately leaves both green while an extra field, a different header, or
#   a divergent body shape reintroduces the existence oracle. Compare the whole
#   response, ignoring what legitimately varies.
response == {{FORBIDDEN_RESPONSE}} ignoring [header.Date, header.X-Trace-Id]
```

**Notes**

`@serial`, and declared as an ordered chain with OT1: this scenario consumes OT1's captured
response, so it must run on the same worker, after it.

**Known gap, stated rather than omitted:** a strict implementation would also compare
response *timing*, since a constant-time-versus-lookup difference leaks the same
information. A scenario runner cannot measure that reliably, so it is out of scope here and
belongs to a timing-analysis tool.

---

## OT3 — Cancelling another tenant's order is refused and has no effect

**Behavior:** a cross-tenant cancellation is refused, the order is unchanged, and no
cancellation event is published.
**Provenance:** @intent — SEC-7
**Tags:** @orders @security @escape-hatch @side-effect @destructive

```requires
fixture: tenant_a_token
fixture: tenant_b_token
seed: order owned by tenant_b     # escape-hatch WRITE (mutable environments only)
```

```setup
# Seed-visibility control, exactly as OT1. method/12 A4.
request GET {{surface(api)}}/v1/orders/{{FOREIGN_ORDER_ID}}
        Authorization: Bearer {{TENANT_B_TOKEN}}
expect  status == 200
        body.data.status == "pending"

# A second tenant-B order, used only to drive the control path below.
request POST {{surface(api)}}/v1/orders
        Authorization: Bearer {{TENANT_B_TOKEN}}
        { "customer_id": "{{TENANT_B_CUSTOMER}}",
          "items": [{ "sku": "SKU-CATALOGUE-STATIC", "quantity": 1 }] }
expect  status == 201
capture CONTROL_ORDER_ID = response.body.data.id
```

```control
# ── PROXY CONTROL ─────────────────────────────────────────────────────────────────
# This scenario asserts that NO event was published, which inverts the fail-closed
# default: a match is FAIL, no match through the budget is PASS.
#
# The control CANNOT use the absence predicate. The absence predicate pins
# {{FOREIGN_ORDER_ID}}, and the only thing that could publish a matching event is the
# bug under test. So this is a PROXY control (method/06): it traverses the identical
# topic → subscription → sink path with a different key.
#
# What it proves:  the publish path is live end-to-end.
# What it does NOT prove: routing for {{FOREIGN_ORDER_ID}} specifically. A dead
#   partition or a mis-scoped subscription filter would pass both this and the
#   absence check. That residual gap is stated here deliberately.
#
# There is NOTHING TO DRAIN. The control event can never match the absence predicate,
# so count-of-survivors does not apply. An earlier version drained it and described
# this as a survivor count, which was incoherent.
request POST {{surface(api)}}/v1/orders/{{CONTROL_ORDER_ID}}/cancel
        Authorization: Bearer {{TENANT_B_TOKEN}}
expect  status == 200
expect  event(billing).published where order_id == {{CONTROL_ORDER_ID}}
        within cross_service
        exists
```

<!--
A failed control is FAIL (reason: control), never INFRA. The system WAS exercised and
did not do what the scenario required — tenant B cancelling its own order should
publish. Filing that as environment noise takes a product regression and puts it
somewhere nothing blocks on. method/02.
-->

```request
POST {{surface(api)}}/v1/orders/{{FOREIGN_ORDER_ID}}/cancel
Authorization: Bearer {{TENANT_A_TOKEN}}
```

```expect response
status == 404
body.code == "ORDER_NOT_FOUND"
```

```expect side-effect
# Unchanged, via a product read (method/12 B7 — a store read can be served stale).
request GET {{surface(api)}}/v1/orders/{{FOREIGN_ORDER_ID}}
        Authorization: Bearer {{TENANT_B_TOKEN}}
expect  status == 200
        body.data.status == "pending"

# Poll the FULL budget. Any match is FAIL.
event(billing).published where any_field contains {{FOREIGN_ORDER_ID}}
  absent through cross_service
```

```control
# ── BRACKET ───────────────────────────────────────────────────────────────────────
# Re-assert the control path after the absence poll. A control only at the front
# proves the pipeline was live when the window opened; it says nothing about a
# pipeline that died during the poll — which would false-green the absence check.
request POST {{surface(api)}}/v1/orders/{{CONTROL_ORDER_ID_2}}/cancel
        Authorization: Bearer {{TENANT_B_TOKEN}}
expect  status == 200
expect  event(billing).published where order_id == {{CONTROL_ORDER_ID_2}}
        within cross_service
        exists
```

```teardown
store(orders).delete({{FOREIGN_ORDER_ID}})      # escape-hatch (mutable only)
store(orders).delete({{CONTROL_ORDER_ID}})      # escape-hatch (mutable only)
store(orders).delete({{CONTROL_ORDER_ID_2}})    # escape-hatch (mutable only)
```

**Notes**

The absence predicate is deliberately **broad** — `any_field contains
{{FOREIGN_ORDER_ID}}` rather than `order_id == …`. Breadth inverts for absence
assertions ([`method/03`](../../method/03-evidence-rules.md) R3): a narrow pin creates
misses, so a bug publishing the id in a differently-named field would slip past and the
scenario would go green. The order id is unique to this execution, so breadth costs nothing.

`{{CONTROL_ORDER_ID_2}}` is a third tenant-B order created in setup for the bracket; it is
elided above for brevity in the same way `SKU-CATALOGUE-STATIC` is — see
[`../guide.md`](../guide.md) §7 for the fixture list.

---

## OT4 — An expired credential is refused

**Behavior:** a structurally valid but expired token is rejected, distinguishably from a
malformed one.
**Provenance:** @contract
**Tags:** @orders @security @escape-hatch

```requires
fixture: tenant_a_token
fixture: tenant_a_customer
seed: expired token for tenant_a    # escape-hatch — no endpoint mints an expired token
```

```setup
# A real order to request, so the 401 is demonstrably credential-driven rather than a
# 404 that happens to share a status family.
request POST {{surface(api)}}/v1/orders
        Authorization: Bearer {{TENANT_A_TOKEN}}
        { "customer_id": "{{TENANT_A_CUSTOMER}}",
          "items": [{ "sku": "SKU-CATALOGUE-STATIC", "quantity": 1 }] }
expect  status == 201
capture TARGET_ORDER_ID = response.body.data.id
```

```request
GET {{surface(api)}}/v1/orders/{{TARGET_ORDER_ID}}
Authorization: Bearer {{EXPIRED_TOKEN}}
```

```expect response
status == 401
body.code == "TOKEN_EXPIRED"
```

```teardown
request DELETE {{surface(api)}}/v1/orders/{{TARGET_ORDER_ID}}
        Authorization: Bearer {{TENANT_A_TOKEN}}
expect  status == 204
```

**Notes**

`TOKEN_EXPIRED` is deliberately distinct from `UNAUTHORIZED`. Asserting the distinction is
what keeps it from being accidentally collapsed during a refactor of the auth layer.

**Privilege note — read this before copying the pattern.** Minting an expired token usually
requires access to the **signing key**, which is a substantially heavier grant than a store
write: it is the ability to forge any credential in the environment. Two consequences: this
fixture belongs only in an environment where that key is disposable, and it should never be
provisioned by the same identity that runs the rest of the suite. If your key is shared with
anything you care about, prefer shortening the token TTL in the environment's configuration
and waiting it out. See [`method/10`](../../method/10-environments.md) → Executor privilege.
