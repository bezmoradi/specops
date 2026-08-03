---
# ── Header table: THIS IS THE BEHAVIORAL CONTRACT ─────────────────────────────
# Every field below is observable by a caller. A stale field is silent contract
# drift, not a cosmetic typo. See method/04-spec-altitude.md.
id: <area>-<action>              # matches the filename; stable, referenced by other specs
method: POST                     # or: transport: queue | schedule | signal
path: /v1/<resource>
surface: api                     # logical surface — resolved by environments/<env>.yaml
auth: none                       # none | bearer | bearer(admin) | api-key | <scheme>
tier: <lane tier>                # rate-limit tier, or a lane name for non-HTTP transports
envelope: default                # which wire shape this surface speaks
tags: [<area>, regression]
---

# <METHOD> <path>

**Conventions:** see [guide.md](./guide.md).

<!--
Preamble. Non-normative — it explains, it never decides a verdict.

State the request body by its REAL wire keys, types, and which are required:

  - `customer_id` (string, required)
  - `items[]`     (array, required, min 1)

Do NOT name internal types, handlers, files, or physical table names.
Logical store names — store(orders) — are fine; they bind per environment.
See method/04-spec-altitude.md.
-->

---

## <ID>1 — <short imperative title>

**Behavior:** <one line: who does what → observable outcome>
**Provenance:** @intent — <requirement ref, REQUIRED for @intent>  |  @contract
**Tags:** @<area> @smoke|@regression [@security] [@side-effect] [@destructive] [@serial]

```requires
fixture: <name>            # absence → BLOCKED, computed, never judged
credential: <name>
```

```setup
# Steps before the stimulus. MAY capture and MAY assert.
# Anything that decides the verdict belongs here, NOT in prose. method/03.
# A failed setup assertion is FAIL (reason: setup), never INFRA.
UNIQUE = mint(slug)
request POST {{surface(api)}}/v1/<prerequisite>
        Authorization: Bearer {{TOKEN}}
        { "name": "{{UNIQUE}}" }
expect  status == 201
capture PREREQ_ID = response.body.data.id
```

```request
POST {{surface(api)}}/v1/<resource>
Authorization: Bearer {{TOKEN}}
Content-Type: application/json

{
  "customer_id": "{{CUSTOMER_ID}}",
  "items": [{ "sku": "{{UNIQUE}}", "quantity": 2 }]
}
```

```expect response
status == 201                                    @intent
body.code == "<STABLE_CODE>"                     @contract
body.data.id exists                              @contract
body.data.status == "<exact literal>"            @contract
body.data.items count == 1                       @contract
```

```capture
RESOURCE_ID = response.body.data.id
TRACE_ID    = response.header["X-Trace-Id"]
```

```expect side-effect
# Independent verdict. A 2xx never excuses an absent effect.
# Pin a PER-EXECUTION-UNIQUE correlator. Never a time window.
# `within` = appearance (first true wins). `hold` = invariant (true at expiry).
# Anything shaped "still equals" / "exactly N" / "unchanged" MUST use `hold`.

store(<resource>).item({{RESOURCE_ID}})                      within index_propagation
  .status == "<exact literal>"

# EXACTLY-ONCE needs BOTH lines. `exists` is true of one event and equally true of two;
# `count == 1 within` is first-true-wins and stops looking before a delayed duplicate
# lands. Receipt bounds the count from nothing; bracket bounds it forward in time.
# The bracket's budget covers the RETRY interval. method/12 H1.
event(<topic>).published where resource_id == {{RESOURCE_ID}}  within cross_service
  count == 1
  .event_type == "<exact literal>"

event(<topic>).published where resource_id == {{RESOURCE_ID}}  hold single_hop
  count == 1
```

```deviation
# OPTIONAL — only when the asserted behavior contradicts a documented intent.
# The scenario asserts `observed` above, so it does not rot. This block records what the
# contract says, and the evaluator resolves three ways: actual satisfies `intent` -> PASS
# plus a `deviation_resolved` finding (fixing the bug does NOT turn this red); satisfies
# `observed` -> KNOWN-DEVIATION, counted separately, never folded into PASS; neither ->
# FAIL, reason `deviation_drifted`. A citation is required. method/02.
assertion: status
intent:    == <intended>   cite: <resolvable reference>
observed:  == <actual today>
```

```teardown
# MANDATORY for anything that mutates state.
# Prefer the product's own delete endpoint — a 2xx is a free extra assertion.
request DELETE {{surface(api)}}/v1/<resource>/{{RESOURCE_ID}}
        Authorization: Bearer {{TOKEN}}
expect  status == 204
```

**Notes** <!-- optional -->

<!--
Keep a note ONLY if deleting it would let a maintainer silently weaken the test:
test-design rationale · a non-obvious status a reader would assume is a bug ·
an anti-enumeration property · an idempotency or ordering caveat.

CUT: changelog entries · internal symbol names · "verified live <date>" stamps
(those belong in RUNLOG.md, where they will be read in context).
-->

---

## <ID>2 — Unauthenticated request is rejected

**Behavior:** a request with no credential is rejected before any work occurs.
**Provenance:** @contract
**Tags:** @<area> @security @no-auth

```request
POST {{surface(api)}}/v1/<resource>
Content-Type: application/json

{ "customer_id": "any" }
```

```expect response
status == 401
body.code == "UNAUTHORIZED"
```

---

## <ID>3 — A resource belonging to another tenant is not reachable

**Behavior:** a caller cannot read, mutate, or detect a resource outside their tenancy.
**Provenance:** @intent — <requirement ref>
**Tags:** @<area> @security @escape-hatch @destructive

```requires
fixture: caller_token             # the caller who must be refused
fixture: owner_token              # the legitimate owner — needed for the control below
seed: foreign_resource            # escape-hatch WRITE (mutable environments only)
```

```setup
# ── SEED-VISIBILITY CONTROL — load-bearing. method/12 A4. ─────────────────────
# The seed is a raw store write. If it did not populate what the application's read
# path needs, the product returns "not found" for EVERYONE, the assertions below
# pass, and the protection under test was never exercised.
# Prove the PRODUCT can see it, using the OWNER's credential, first.
request GET {{surface(api)}}/v1/<resource>/{{FOREIGN_RESOURCE_ID}}
        Authorization: Bearer {{OWNER_TOKEN}}
expect  status == 200
        body.data.id == "{{FOREIGN_RESOURCE_ID}}"
```

```request
GET {{surface(api)}}/v1/<resource>/{{FOREIGN_RESOURCE_ID}}
Authorization: Bearer {{CALLER_TOKEN}}
```

```expect response
# Assert the anti-enumeration property deliberately: the response for "exists but
# forbidden" and "does not exist" MUST be indistinguishable, or the endpoint is an
# existence oracle. Capture this response and compare it in a sibling scenario that
# requests a genuinely absent id.
status == 404
body.code == "NOT_FOUND"
```

```capture
FORBIDDEN_RESPONSE = response
```

```expect side-effect
# Prove the foreign resource SURVIVED — a destructive scoping bug would satisfy the
# response assertion above. Use a PRODUCT read, not a store read: a store read can be
# served stale and show a row that has already been destroyed. method/12 B7.
request GET {{surface(api)}}/v1/<resource>/{{FOREIGN_RESOURCE_ID}}
        Authorization: Bearer {{OWNER_TOKEN}}
expect  status == 200
        body.data.status == "<unchanged value>"
```

```teardown
store(<resource>).delete({{FOREIGN_RESOURCE_ID}})   # escape-hatch (mutable only)
```
