# 07 — Isolation

A suite of a few hundred scenarios run serially takes too long to be useful. Run in
parallel against a shared environment, scenarios poison each other — and the poisoning
mostly produces **false greens and unexplained reds**, not clean failures.

This page is about running many scenarios concurrently without them lying to each other.

## The three collision surfaces

| Surface | Symptom | Fix |
|---|---|---|
| **Shared throttling budget** | Unrelated scenarios receive throttled responses; assertions fail for reasons unrelated to the behavior | Partition by throttling bucket |
| **Shared fixtures** | One scenario mutates or deletes state another is asserting on | Per-scenario subjects; a reserved pool for shared ones |
| **Shared evidence streams** | One scenario's evidence matches another's predicate | Per-execution correlators ([`06-side-effects.md`](06-side-effects.md)) |

The third is the dangerous one, because it produces green.

## Partition by throttling bucket

Rate limiters typically key on client identity — usually source address. Every worker on
one machine shares one bucket. Two concurrent workers exercising the same throttled
category will trip each other's limits, and the resulting failures look like product
defects.

**If your limiter has independent buckets per category, the natural unit of parallelism is
one lane per category.**

Derive lanes from the specifications themselves. Each specification's header table already
declares its rate-limit tier ([`04-spec-altitude.md`](04-spec-altitude.md)), so the
partition is computable, not hand-maintained:

```
lane 01  auth          →  the specifications declaring tier: auth
lane 02  read          →  the specifications declaring tier: read
lane 03  mutation      →  …
```

Two consequences worth planning for:

- **A lane is one worker, and files within it run in order.** That property is what makes
  ordered dependencies safe (below).
- **Splitting one category across multiple workers does not multiply its budget.** Same
  source, same bucket. Sub-splitting only helps once the bucket has genuine headroom.

## Quarantine the floods

Any scenario that deliberately exhausts a limit — proving that throttling engages — burns
the entire budget for that category for the full window. It cannot run concurrently with
anything sharing that bucket.

Mark these `@serial` and run them **alone**, in their own phase, typically last. Honor the
cooldown afterward.

When choosing what to flood, pick a **read** operation that mutates nothing. Flooding a
mutating category creates hundreds of real resources, and flooding a category where each
request has an external side effect — registering something with a third party, sending
mail — creates a mess that outlives the run.

## Ordered dependencies

Some scenarios genuinely depend on a sibling: one creates the subject, a later one asserts
a delayed side effect of that creation and reuses its captured identifiers.

Mark the whole chain `@serial` and require the runner to keep it on **one worker, in file
order, never interleaved**.

Prefer avoiding these. A chain is a scenario spread across several entries, and it fails in
ways that are hard to attribute — when the third link fails, the cause is often the first.
Use them where the alternative is genuinely worse, such as when re-creating the subject
would be slow or would itself perturb what you are measuring.

## Fixtures

Three categories, and mixing them up is the most common source of cross-scenario damage:

| Category | Lifetime | Rule |
|---|---|---|
| **Permanent** | Survives every run | Never mutated, never deleted, hardcoded into teardown's keep-list |
| **Run-scoped** | Provisioned at run start, destroyed at run end | Shared read-only across scenarios; mutated only by scenarios that own them |
| **Scenario-scoped** | Created and destroyed within one scenario | Uniquely named per execution; anything destructive uses these |

**A destructive scenario must never consume a run-scoped fixture.** It mints its own,
uniquely named, and destroys it in teardown. The moment a destructive scenario deletes a
shared subject, every sibling asserting on that subject fails, and the reported cause will
be wrong.

## Clean slate, or a maintained pool

Two viable lifecycle policies. Choose deliberately and write it down.

**Clean slate.** Provision everything fresh at run start; destroy everything at run end.
The environment returns to a known set of permanent fixtures every time.

- Scenarios asserting *freshness* — a new subject's initial version, an empty collection, a
  never-touched flag — are only honest against freshly provisioned fixtures.
- Listing endpoints stay assertable, because leftover fixtures do not accumulate and make
  counts unpredictable.
- The next run, and a colleague's run, start from the same zero.
- **Cost:** provisioning time on every run.

**Maintained pool.** Keep fixtures standing between runs.

- Faster.
- **Cost:** the pool drifts. Versions increment, credentials rotate, scenario-scoped
  leftovers accumulate. Freshness assertions quietly stop meaning what they say.

Clean slate is the right default. Its cost is measured in minutes and is paid in
determinism.

Whichever you choose, the teardown must **verify** it worked — re-list at the end and
assert the environment matches the expected steady state. A teardown that silently
half-completes is how a pool starts drifting under a clean-slate policy.

## Teardown discipline

1. **Mandatory for anything that mutates state.** No exceptions.
2. **Prefer the product's own delete endpoint.** A `2xx` from it is a free extra
   assertion; a broken delete surfaces as a finding rather than hiding.
3. **Direct store deletion is the last resort.** Use it for state no endpoint can reach, and
   as a best-effort safety net after a failure path.
4. **Delete the reverse mappings too.** Uniqueness sentinels, secondary index entries,
   reverse-lookup rows. A missed one can permanently reserve an identifier and make a future
   run fail for reasons nobody can explain.
5. **When the delete endpoint is the thing under test, the lifecycle is the test.** Create,
   delete, confirm gone — leave it.

## Guard before you mutate

Check credentials and preconditions **before** the first write, not lazily as each is
needed. A credential that expires halfway through a multi-step mutation leaves partially
applied state and orphans, and the resulting environment damage outlives the run.

## Per-execution uniqueness

Every scenario that creates something names it uniquely per execution, and every event
published carries a freshly minted identifier.

This matters beyond collision avoidance. Many systems deduplicate on an identifier. Reusing
one across runs means the second run's stimulus is silently a no-op — the system correctly
ignores a duplicate, the scenario observes no new effect, and depending on which way the
assertion points, you get either a mysterious red or a permanent green that verifies
nothing.
