# Execution contract — <system name>

<!--
Copy this to specs/guide.md in your own repository and fill it in.

This file answers a question the specifications deliberately do NOT answer:
"what counts as having observed something, HERE?"

Specifications state WHAT should be true. This states what evidence is acceptable
proof of it in this system. Without it, every agent invents its own answer and no
two runs mean the same thing.

Method reference: https://github.com/bezmoradi/specops — method/01-overview.md
-->

## 0 — What is different about this system

<!--
The 3–6 facts an agent must know before writing or running anything here, that it
could not infer. Examples of the KIND of fact that belongs here:

  - This system has N entry points; which paths each serves; what happens if you
    send a request to the wrong one.
  - It issues no credentials of its own; the bootstrap authenticates elsewhere.
  - It has surfaces that deliberately do NOT use the house response envelope.
  - Its "request" is sometimes an event publish, and the verdict is the side effect.
  - Which of its behaviors are asynchronous, and which are not.
-->

---

## 1 — Verdicts

This suite follows [`method/02-verdicts.md`](https://github.com/bezmoradi/specops/blob/main/method/02-verdicts.md):
`PASS` · `FAIL` · `BLOCKED` · `N/A` · `INFRA` · `INDETERMINATE`.

Local additions or clarifications:

<!-- e.g. which scenarios are legitimately N/A here, and why -->

---

## 2 — The rules, restated as local commitments

These are non-negotiable and are copied here so a reader of *this* repository sees
them without following a link:

1. **The agent never votes.** Verdicts are computed from recorded evidence.
2. **Fail closed.** Missing, empty, or errored evidence is `FAIL`.
3. **No empty captures, no weaker correlator.**
4. **Bounded poll, never re-run-until-green.**
5. **Altitude:** no internal symbol, file, or line number appears in a specification.
6. **Provenance:** every assertion is tagged `@intent` or `@contract`.

---

## 3 — Correlators

| Situation | Correlator | Where it appears |
|---|---|---|
| Request-initiated side effect | <trace identifier> | <where it is observable> |
| Suite-published event | the minted `event_id` | <where> |
| Newly created resource | the returned id | <where> |
| Per-scenario subject | <unique naming convention> | <where> |

**Known gaps.** <!--
Identifiers the system deliberately does NOT emit (personal data, for instance),
and what the correct bridge identifier is instead. Getting this wrong produces a
permanent red that looks like a product failure.
-->

---

## 4 — Evidence sources, ranked

Per [`method/06-side-effects.md`](https://github.com/bezmoradi/specops/blob/main/method/06-side-effects.md):
state and correlated events are authoritative; counters are corroboration only.

| Source | Logical name | How it is read here | Authoritative? |
|---|---|---|---|
| Persisted state | `store(<name>)` | | yes |
| Application logs | `log(<service>)` | | yes, when correlated |
| Published events | `event(<topic>)` | | yes, when pinned |
| Counters | `metric(<name>)` | | corroboration only |

**Replica count:** <N>. Evidence for one execution lands on **one** replica — always
aggregate across all of them.

**Private sinks:** <!-- whether the suite subscribes its own consumer to any topic, and how -->

---

## 5 — Poll budgets

Defined per environment in `environments/<env>.yaml`, referenced by name from
specifications. Budget classes in use:

| Class | Covers |
|---|---|
| `index_propagation` | Secondary index / read-replica consistency |
| `single_hop` | One asynchronous delivery |
| `cross_service` | Producer → transport → consumer |

---

## 6 — Authentication bootstrap

<!--
The exact sequence to obtain each credential the suite needs, and how sessions are
extended without repeating the expensive step. State which steps are automated and
which — if any — require a human, and be honest about it: a "human-in-the-loop"
step that has quietly been automated should be re-documented, and one that has not
blocks unattended runs.
-->

---

## 7 — Fixtures

| Fixture | Category | Provisioned by | Torn down by |
|---|---|---|---|
| | permanent / run-scoped / scenario-scoped | | |

**Lifecycle policy:** clean-slate | maintained pool — <state which, and why>

**Steady state:** after teardown this environment contains exactly: <list>. Teardown
verifies this and fails if it does not hold.

---

## 8 — Escape hatches

Permitted only for state no endpoint can reach, only in `requires` and `teardown`,
only in a `mutable` environment, and never as evidence of product behavior.

| Hatch | Why no endpoint reaches this state |
|---|---|

---

## 9 — Parallelism

**Partition:** <how lanes are derived — usually one per throttling category>

**Serial / quarantined:** <which scenarios must run alone, and why>

**Ordered chains:** <which scenarios depend on a sibling, and in what order>

---

## 10 — Running

```
# static gate — no system, no model, seconds
<command>

# provision
<commands>

# run
<commands>

# teardown, and verify the steady state
<commands>
```

---

## 11 — Local false-green traps

<!--
The system-specific ways THIS suite has reported green while broken. This is the
most valuable section of this file. Start it empty and add to it after every
incident a green run failed to catch.

Format: symptom → mechanism → rule → cost.
General catalogue: method/12-false-greens.md
-->
