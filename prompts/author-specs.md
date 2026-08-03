# Prompt — author specifications

Give this to your coding agent, together with one endpoint or feature to cover.

Read [`method/04-spec-altitude.md`](../method/04-spec-altitude.md) and
[`method/05-provenance.md`](../method/05-provenance.md) first if you want to understand
why the constraints below are what they are.

---

## Prompt

> You are writing a **behavioral specification** for `<ENDPOINT OR FEATURE>`.
>
> A specification is not a test script. It is a contract stating what the system does,
> observably, from outside. It will be executed later by a different agent against a
> deployed environment.
>
> **Use the skeleton in `specs/spec.md`. Follow the conventions in `specs/guide.md`.**
>
> ### Altitude — the hard constraint
>
> Your specification must survive a behavior-preserving refactor. If someone renames every
> class, splits every module, and swaps the persistence layer without changing observable
> behavior, your specification must still be valid and still pass.
>
> Therefore it must **never** contain:
> - class, function, handler, or method names
> - file paths or line numbers
> - table names, index names, or key shapes
> - internal type or DTO names
> - framework or library names
>
> You **may and should** read the implementation to learn exact facts — status codes,
> field names, stable response codes. Record the **fact**, never the **citation**:
>
> ```
> ✗  returns 409 (see OrderService.create, order_service.go:212)
> ✓  a duplicate customer reference returns 409 with code ORDER_ALREADY_EXISTS
> ```
>
> Describe request and response bodies by their **real wire keys** and types, with the
> real casing. If the field is `customer_id`, write `customer_id` — `customerId` is a
> different key that a strict decoder rejects.
>
> ### Mode — read this first
>
> **`MODE: brownfield`** — the system exists. Verify every code, status, and field name
> against it or its published contract. **Never invent one.**
>
> **`MODE: greenfield`** — the system does not exist yet, and this specification is being
> written *before* the implementation. There is nothing to verify against, so:
> - Values come from the **requirement**, and are tagged `@intent` with a citation.
> - Where the requirement is silent — the exact stable code token, the error body shape —
>   **declare** a value, mark it `@contract (proposed)`, and say plainly in the preamble
>   that the implementation must conform to it or the specification must be updated when it
>   does not.
> - The specification is **binding on the implementation**, not a description of it. That is
>   the whole point of writing it first, and it is the only way `@intent` assertions get
>   created at scale.
>
> Default to `brownfield` if I did not say. "Never invent a code" applies to `@contract`
> assertions about an existing system; it does not apply to a greenfield declaration.
>
> ### Provenance — tag every assertion
>
> Mark each assertion:
> - `@intent` — derived from a requirement, ticket, or design document. **A citation is
>   mandatory.** `@intent` with no reference will fail the static gate.
> - `@contract` — derived from reading the implementation or observing the running system.
>
> Be honest. If you learned a value by reading the code, it is `@contract` even if it
> feels obviously correct.
>
> If you cannot say where a value came from — if it merely "seemed right" — **do not
> assert it.** Ask me instead. An untraceable assertion passes or fails by coincidence.
>
> ### Assertions
>
> - **Assert exact literals** wherever the value is deterministic. `is string` where the
>   value is known to be `"ACTIVE"` proves almost nothing.
> - **Assert stable machine-readable codes**, not human-readable message copy. Message
>   text gets reworded; a specification that fails on a copy edit will be weakened until
>   it fails on nothing.
> - **Never invent a code or status.** Verify each one against the real system or its
>   published contract.
>
> ### Side effects
>
> If this endpoint or feature has downstream effects — a row written, an event published,
> a notification dispatched, a counter moved — assert them in a **separate**
> `expect side-effect` block. The response and the side effect are independent verdicts.
> A `2xx` never excuses an absent effect.
>
> Every side-effect assertion must:
> - pin a **correlator unique to this execution** — a minted identifier, a propagated
>   trace, a freshly created resource id, a per-run unique subject. **Never a time
>   window.**
> - declare a **poll budget** by name from `environments/`, not a literal duration.
>
> Rank evidence: persisted **state** and **correlated events** are authoritative;
> **counters** are corroboration only and are asserted as "moved", never as an exact `+1`.
>
> Reading persisted state is **evidence**, not an escape hatch. Writing state the product
> cannot produce is the escape hatch, and it belongs only in `requires` and `teardown`.
>
> ### Pick the right temporal wrapper — this is where specifications most often fail
>
> - `within <budget>` — **appearance.** Poll until true; first true evaluation wins.
> - `hold <budget>` — **invariant.** True at every sample and at budget expiry.
> - `absent through <budget>` — **inverted.** Any match fails.
>
> **Anything shaped "still equals", "exactly N", or "unchanged" MUST use `hold`.** With
> `within` it is satisfied the instant it is first true — which for a count that must not
> grow is *before the system could have done the wrong thing*. That is how an idempotency
> scenario comes to pass against every double-writing system.
>
> And `hold` alone does not prove the stimulus arrived. Pair it with evidence of receipt — a
> dedup-hit log line, a processed counter, a consumer acknowledgement — or zero-delivery
> satisfies the invariant trivially.
>
> ### Negative scenarios — write these, they are the point
>
> For anything authenticated or tenant-scoped, cover:
>
> - **unauthenticated** — no credential.
> - **cross-tenant** — a resource belonging to someone else. Two things are load-bearing
>   and both look deletable:
>   1. **Seed a real foreign resource.** Without it the request fails identically whether or
>      not isolation works, and the scenario can never fail.
>   2. **Prove the product can see the seed** — the legitimate owner reads it through the
>      API and gets `200`, in a `setup` block, *before* the cross-tenant attempt. A raw
>      store write can be invisible to the application, in which case it returns "not found"
>      for everyone and the scenario passes for the wrong reason.
>
>   Then assert the foreign resource **survived**, via a product read rather than a store
>   read — a store read can be served stale and show a row that was already destroyed.
> - **the existence oracle** — the response for "exists but forbidden" and "does not exist"
>   must be indistinguishable. Capture the first response and **compare the whole response**
>   in the sibling scenario, ignoring volatile headers. Asserting three fields in each
>   leaves both green while an extra field reintroduces the oracle.
> - **malformed input** — where the system makes a documented promise about it.
>
> ### Scenarios asserting something must NOT happen
>
> These invert the default and are the easiest to get silently wrong. Three requirements:
>
> 1. **A subject unique to this execution.** Never a shared one.
> 2. **A broad predicate over a minted marker.** Breadth *inverts* for absence: a narrow pin
>    creates misses, because a bug emitting the value under a different field name slips
>    past. Mint a marker into the stimulus and match it anywhere in the event.
> 3. **A `control` block** proving the observation path is live — of the right kind:
>    - *same-subject* if the subject can legitimately emit this event type (identical
>      predicate; capture and drain);
>    - *proxy* if the target event is producible only by violating the property under test
>      (different predicate; nothing to drain; **state the residual gap**).
>
>    Bracket it — re-assert the control after the absence poll — or state the residual
>    window.
>
> ### Preconditions and teardown
>
> - Declare every precondition in a `requires` block. Missing preconditions produce
>   `BLOCKED`, computed from what you declare — so declare them all.
> - Any scenario that mutates state **must** have a `teardown` block. Prefer the
>   product's own delete endpoint; a `2xx` from it is a free extra assertion.
> - Anything created must be **uniquely named per execution**.
>
> ### Structure
>
> - One specification per endpoint or per event-driven feature, containing all its
>   scenarios.
> - Stable two-letter scenario ids (`OR1`, `OR2`, …) — other specifications will
>   reference them.
> - Fill the header table completely. Every field in it is contract; a stale field is
>   silent drift.
> - Prose is **non-normative**. It explains. **Anything that decides a verdict goes inside a
>   typed block** — including setup steps that capture a value, and controls that must fire.
>   If you find yourself writing "**Setup.** Create an order and capture its id," that is a
>   `setup` block, not a paragraph.
>
> The blocks available to you: `requires` · `setup` · `control` · `request` · `capture` ·
> `expect response` · `expect side-effect` · `teardown`. Use only operators defined in
> `method/03-evidence-rules.md`; if you need one that is not there, say so rather than
> inventing it.
>
> ### If you cannot verify something
>
> Say so. Do not guess a status code, invent a response shape, or assume a field name.
> A specification asserting something plausible-but-wrong is worse than one with a gap,
> because the gap is visible and the wrong assertion is not.
>
> If a behavior has **no observable consequence** — nothing in any response, store, log,
> queue, or metric — say that too. It cannot be verified, and the right fix is often to
> add the observability rather than to write a specification that pretends.

---

## After the agent responds

Review against [`adoption/checklist.md`](../adoption/checklist.md). The two questions that
catch the most problems:

1. **"Where did this number come from?"** — for every status code, limit, and threshold.
   "It seemed right" means send it back.
2. **"Would this still pass if I renamed every class in the service?"** — if not, it is
   over-specified.
