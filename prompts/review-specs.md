# Prompt — review specifications

Give this to an agent — ideally a **different** one from the author — or use it yourself
as a review checklist.

Reviewing specifications is higher-leverage than reviewing test code, because a weak
specification does not fail. It passes, forever, verifying less than everyone believes.

---

## Prompt

> You are reviewing behavioral specifications. **You are looking for specifications that
> will pass while proving nothing.** A specification that is too strict fails loudly and
> gets fixed. One that is too weak is silent, and silence is what you are hunting.
>
> Read [`method/12-false-greens.md`](../method/12-false-greens.md) first — it is the
> catalogue of the failure modes you are looking for.
>
> For each specification, work through the following. Report findings with the scenario
> id, the specific line, and which numbered check it violates.
>
> ### 1 — Can this scenario fail?
>
> The first and most important question. For each scenario, construct the broken system
> that would still pass it.
>
> Flag hard where:
> - A negative or isolation scenario never seeds the resource it claims is unreachable.
>   The request would fail identically whether or not the protection works.
> - An absence assertion has no positive control, so a dead observation path is
>   indistinguishable from correct behavior.
> - The only assertions are on a status code that the endpoint returns for many reasons.
> - A side effect is asserted against state the scenario itself seeded.
>
> ### 2 — Altitude
>
> Would this still pass if every class were renamed, every module split, and the
> persistence layer swapped, with no observable behavior change?
>
> Flag: class or handler names, file paths, line numbers, table or index names, internal
> type names, framework names, assertions on internal call ordering that a caller cannot
> observe.
>
> ### 3 — Provenance
>
> Is every assertion tagged `@intent` or `@contract`?
>
> For each untagged one, ask **where the number came from**. If the answer is neither a
> requirement nor the observed system — if it merely seemed right — flag it. Untraceable
> assertions pass or fail by coincidence and look exactly like good ones.
>
> ### 4 — Assertion strength
>
> Flag:
> - `is string` / `is number` / `exists` where the value is deterministic and known.
>   Assert the literal.
> - Assertions on human-readable message copy where a stable code exists.
> - A scenario whose only assertion is `status == 2xx`.
> - A response body with fields a client would depend on that are not asserted.
>
> ### 5 — Side effects
>
> Flag:
> - An endpoint with a known downstream effect and no `expect side-effect` block.
> - A side-effect assertion sharing a block with the response assertions — they are
>   independent verdicts and must be evaluated separately.
> - A correlator that is a time window, a message string, or an event type alone.
> - A predicate broad enough to match a **previous** execution's evidence still sitting in
>   a queue or log.
> - A counter asserted as an exact `+1` in an environment anything else touches.
> - A missing or literal poll budget — budgets are named and live in the environment file.
>
> ### 6 — Isolation
>
> Flag:
> - A destructive scenario consuming a shared fixture instead of minting its own.
> - A created resource without a per-execution-unique name.
> - A mutating scenario with no `teardown`.
> - A teardown that does not remove reverse mappings, uniqueness sentinels, or secondary
>   index entries — a missed one can permanently reserve an identifier.
> - A scenario that must run after a sibling but is not marked as an ordered chain.
>
> ### 7 — Preconditions
>
> Flag any dependency — a fixture, a credential, a third-party sandbox, a piece of
> hardware — that the scenario assumes but does not declare in `requires`. An undeclared
> precondition surfaces as a confusing `FAIL` instead of an honest `BLOCKED`.
>
> ### 8 — Structure
>
> Flag:
> - An incomplete header table. Every field in it is contract.
> - Prose that is load-bearing — something outside a typed block that a verdict depends on.
> - Notes that are changelog entries, internal symbol references, or point-in-time run
>   stamps. Those rot; cut them.
> - A referenced scenario id that does not exist.
>
> ### Output
>
> Group findings by severity:
>
> - **Cannot fail** — the scenario passes against a broken system. Highest priority; these
>   are actively harmful because they consume budget to manufacture confidence.
> - **Weak** — it can fail, but proves much less than it appears to.
> - **Brittle** — it will fail for non-defect reasons and will therefore be weakened later
>   into one of the above.
> - **Style** — everything else.
>
> For each finding give the scenario id, the line, the check number, and a concrete
> rewrite. "Assert more" is not actionable; "assert `body.data.status == \"ACTIVE\"`
> rather than `is string`" is.
>
> If a specification is genuinely sound, say so plainly and move on. Do not manufacture
> findings — an over-eager review trains people to ignore reviews, which costs more than
> the findings were worth.
