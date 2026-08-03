<!--
GENERATED — do not hand-edit.

A hand-maintained catalogue of hundreds of specifications drifts within weeks, and it
drifts in the direction of claiming coverage that does not exist. Generate this from the
corpus and gate it in the static check. See method/09-gates.md.

Regenerate with: <your command>
-->

# Test index — <system name>

Maps every endpoint and event-driven feature to its specification in [`specs/`](./).
Conventions: [`guide.md`](./guide.md).

**Corpus**

| | |
|---|---|
| Specifications | <n> |
| Scenarios | <n> |
| Assertions | <n> |
| Provenance | <n> `@intent` (<n>%) · <n> `@contract` (<n>%) |
| Declared `N/A` | <n> |

<!--
The provenance ratio is the most informative number here and belongs above any
coverage figure. A suite that is 100% @contract is a regression net — a legitimate
and useful thing to be, and worth stating plainly. See method/05-provenance.md.

Do NOT publish a coverage percentage as a headline. It counts enumeration, not
verification quality. See method/12-false-greens.md G4.
-->

---

## <Area>

| Method | Path | Spec | Scenarios | Provenance |
|---|---|---|---|---|
| POST | `/v1/orders` | `orders-create.md` | OR1–OR6 | 2 @intent · 11 @contract |
| GET | `/v1/orders/{id}` | `orders-get.md` | OG1–OG4 | 0 @intent · 9 @contract |

## <Area — event-driven>

| Transport | Feature | Spec | Scenarios | Provenance |
|---|---|---|---|---|
| queue | `order.cancelled` cascade | `fanout-order-cancelled.md` | FC1–FC3 | 1 @intent · 5 @contract |

## Cross-cutting

| Surface | Spec | Scenarios | Notes |
|---|---|---|---|
| Rate limiting | `rate-limit.md` | RL1 | ⚠ serial / quarantined |
| CORS | `cors.md` | CP1–CP3 | |

<!-- Generated rows link to the real files; the placeholders above are plain text. -->


---

## Uncovered

<!--
GENERATED from the API schema. Every route with no specification. Listing these is
more useful than a coverage percentage, because each line is an action.
-->

| Method | Path | Since |
|---|---|---|

---

## Known gaps

<!--
Hand-maintained. Structural blind spots this suite does not and cannot cover, so
that a green run is not mistaken for more than it is. Be specific about the
boundary. See method/01-overview.md "Scope boundary".

Examples of what belongs here:
  - concurrency and races (needs coordinated actors)
  - real third-party delivery (verified only to the publish boundary)
  - performance and load
  - input-space coverage
-->
