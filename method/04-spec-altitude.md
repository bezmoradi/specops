# 04 — Spec altitude

> A specification states what the system does, observably, from outside.
> It never states how the system is built.

Altitude is the single property that determines whether a suite is an asset or a liability
two years in.

## The refactor test

**A behavior-preserving refactor must leave every specification valid.**

Rename every class. Split every module. Swap the persistence layer. Change the internal
call graph completely. If the observable behavior is unchanged and a specification breaks,
**the specification was wrong** — it was asserting something that was never part of the
contract.

This is a testable claim about your suite, not an aspiration. Run it: take a real
structural refactor, change nothing observable, and run the suite. Every failure is a
specification defect, and the count is your altitude score.

**One honest carve-out: the persistence layer.** A specification that asserts
`store(orders).item(X).status == "cancelled"` depends on a stored field being called
`status`. Swap the persistence layer and rename that field, and the assertion breaks
without any observable behavior changing. So the claim, stated precisely, is:

> A behavior-preserving refactor leaves every **specification** unchanged. It may require
> re-mapping **evidence bindings**.

That is the correct place for the cost to land, and it is why store fields are addressed
through logical names bound per environment
([`10-environments.md`](10-environments.md)) — a persistence change edits one binding file
rather than a hundred specifications. Prefer product-observable evidence where it exists:
if an endpoint will tell you the order is cancelled, assert that instead of the row. Use
store observation where the product exposes no equivalent, which is common for *absence*,
for cascade completion, and for fields the API deliberately does not return.

It is also the strongest argument for the method when you are asked to justify it. A suite
that survives a full internal restructuring with zero specification edits has demonstrated
something no coverage number can.

## The header table is the contract

Every specification opens with the observable facts that define the endpoint or feature.
These, and only these, are contract:

| Field | Why it is contract |
|---|---|
| Method / path, or transport | The address. Changing it breaks callers. |
| Host or surface | Which entry point serves it, when a system has more than one. |
| Auth | What credential is required. Changing it breaks callers. |
| Rate-limit tier | Determines throttling behavior, which callers observe. |
| Response envelope | Which wire shape this surface speaks, when a system has more than one. |

A stale entry here is **silent contract drift**, not a cosmetic typo. If the rate-limit
tier changed and the table did not, the specification is now asserting the wrong
throttling contract and nobody will notice until a client is throttled unexpectedly.

## What must never appear in a specification

| Forbidden | Why |
|---|---|
| Class, function, handler, or method names | Renamed by refactors; never asserted. |
| File paths and line numbers | Wrong within weeks. Actively misleading once wrong. |
| Database table names, index names, key shapes | Persistence is an implementation choice. |
| Internal type or DTO names | Not on the wire. The wire keys are what callers see. |
| Framework or library names | Replaceable without changing behavior. |
| Queue and topic *implementation* names | Use logical names; bind them per environment ([`10-environments.md`](10-environments.md)). |
| Internal call ordering | Unless externally observable, in which case state the observation, not the ordering. |

**Physical names are forbidden; logical names are not.** A specification may not say
`staging_orders_v2` or name an index. It may say `store(orders)`, which the environment
file binds to whatever that is here ([`10-environments.md`](10-environments.md)).

## Two different things that both touch the store

These are constantly conflated, and conflating them produced a contradiction in an earlier
version of this method. They are separate concepts with separate rules.

| | **Store observation** | **Escape hatch** |
|---|---|---|
| What it is | *Reading* persisted state as evidence of an outcome | *Writing* state the product cannot be made to produce |
| Status | A **sanctioned grey-box evidence channel** | A **narrow, labelled exception** |
| Where it may appear | `expect side-effect` — it is top-tier evidence ([`06`](06-side-effects.md)) | `requires` and `teardown` only |
| May it prove product behavior? | **Yes.** That a row exists, is gone, or changed is the strongest signal available | **Never.** A row you wrote yourself proves nothing |
| Environment | Any | `mutable` only |

Store *observation* is not an escape hatch. This method is explicitly grey-box
([`01-overview.md`](01-overview.md)): the system is exercised from outside and evidence is
gathered from inside. Reading the order row to confirm the order was created is the whole
point.

Store *writing* — manufacturing an expired token, a corrupted row, another tenant's data —
is the exception, and it carries the four conditions in
[`06-side-effects.md`](06-side-effects.md#escape-hatches).

## Use real wire keys

Describe request and response bodies by their **actual JSON keys**, types, and which are
required. Not by the internal type name.

```
✗   body is a CreateOrderRequest with a customer reference
✓   body.customer_id (string, required)
    body.items[] (array, required, min 1)
```

And use the *real* casing. If the API field is `customer_id`, write `customer_id`. Writing
`customerId` is not a stylistic choice — it is a **different key**, one a strict decoder
rejects, and specifications that use it will assert against a field that does not exist.

## The tension, and how to resolve it

There is a real tension between altitude and accuracy:

- To write an accurate assertion you must know the exact status code, the exact field
  name, the exact token. Usually that means reading the implementation.
- But the specification must not *cite* the implementation.

The resolution: **read the implementation to learn the fact; record the fact, not the
citation.**

```
✗   returns 409 (see OrderService.create, order_service.go:212)
✓   a duplicate customer reference returns 409 with code ORDER_ALREADY_EXISTS
```

The second survives the file being deleted. The first becomes a lie the moment the file
moves, and a lie in a specification is worse than a missing detail because it will be
trusted.

This tension has a second consequence, which is the subject of
[`05-provenance.md`](05-provenance.md): an assertion learned by reading the implementation
is verifying that the system still does what it does, not that it does what it should.
That is worth knowing about your own suite.

## Notes discipline

Specifications accumulate notes. Most of them rot. Keep a note only if deleting it would
let a maintainer silently weaken the test.

**Keep:**

- Test-design rationale — why this scenario is structured the way it is.
- A non-obvious status or response shape that a reader would otherwise assume is a bug.
- An anti-enumeration property — why two different failures deliberately return the same
  response.
- An idempotency or ordering caveat that affects how the scenario must be run.

**Cut:**

- Changelog entries. `Fixed 2026-04-02: switched to a transaction` is history; use git.
- Anything naming internal symbols with no bearing on running the scenario.
- Point-in-time run stamps. `Verified live 2026-06-19` belongs in the run ledger, where it
  will be read in context, not in the specification, where it will rot.

## Symptoms of over-specification

Check for these when reviewing:

1. **A rename broke it.** Definitional.
2. **You cannot write the assertion without opening the source.** If the observable
   behavior cannot be stated from outside, either it is not observable — a finding worth
   raising — or you are asserting an implementation detail.
3. **It asserts an intermediate step.** "The record is written before the event is
   published" is only a valid assertion if a caller can observe the ordering. If they
   cannot, you have specified the mechanism.
4. **It asserts a value nobody depends on.** An internal identifier's format, an
   auto-generated field's exact shape. If no client branches on it, changing it is not a
   breaking change and the specification should not say otherwise.
5. **It reproduces a message string verbatim.** Human-readable copy is not contract. Assert
   the stable code.

## The counter-failure: under-specification

Altitude is a ceiling, not a target. Do not respond to this page by asserting only
`status == 200`.

A specification is a contract, and a contract that says almost nothing is satisfied by
almost anything. The correct altitude is: **every fact a client could reasonably depend
on, and nothing else.** When in doubt, ask whether a caller would notice the change. If
yes, assert it. If no, leave it out.
