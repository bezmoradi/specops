# Case 006 — `BLOCKED` is computed from `requires`, in both directions

**Catches:** [`method/12`](../../../method/12-false-greens.md) C2 — `BLOCKED` as a judgement
call. If `BLOCKED` is something an executor may decide, it becomes the drain failures escape
through, and the blocked pile is the one pile nobody triages.

**The distinction under test.** `BLOCKED` is a function of the specification's declared
`requires` and the recorded availability of each requirement. Nothing else. That constrains
the evaluator in **two** directions, and most implementations only get the first:

- A scenario whose `requires` is unmet is `BLOCKED` **even if everything else passed**.
- A scenario whose `requires` is met is **not** `BLOCKED`, however awkward the run was.

The second direction is the one that matters, because it is the direction that hides
failures.

---

## The specification under test

```requires
fixture: tenant_b_seeded_order
credential: admin_token
```

```expect response
status == 404
body.code == "ORDER_NOT_FOUND"
```

---

## Fragment A — a requirement is unavailable

Against `requirements` in the bundle: `tenant_b_seeded_order` recorded `available: false`.
The executor ran nothing further.

Expected: **`BLOCKED`**, reason `tenant_b_seeded_order` — the **name of the requirement**,
not a sentence. The two response assertions are reported `evaluated: false`.

A reason that reads "could not set up the test" is not usable: nobody can tell from it
whether the fixture is missing, the seeder is broken, or the environment is read-only. The
name can be grepped, counted, and assigned.

## Fragment B — a requirement is unavailable, and it passed anyway

Same unmet requirement. But this bundle records the full scenario having run, with both
response assertions `true`.

Expected: **`BLOCKED`**, reason `tenant_b_seeded_order`. **Not `PASS`.**

Ordering step 1 stops before anything else is considered, and this fragment is why the stop
is unconditional rather than "unless it worked out." A `404` observed with no seeded order
present is exactly the reading a system with **no tenancy enforcement at all** produces — see
[`method/12`](../../../method/12-false-greens.md) A4. The assertions are true and they are
about nothing. An evaluator that promotes them to `PASS` has converted a missing fixture into
a clean bill of health for the property the scenario exists to check.

## Fragment C — `BLOCKED` declared with every requirement met

Both requirements recorded `available: true`. The executor recorded
`"verdict_hint": "BLOCKED"` with the note *"could not locate the order id in the response,
skipping"* and issued no request.

Expected: **`FAIL`**, reason `assertion_not_evaluated`, **plus** harness finding
`blocked_without_unmet_requirement`.

This is C2 itself. Every declared precondition was available, so `BLOCKED` is unreachable;
the scenario simply did not run, and a scenario that did not run is not a scenario that was
prevented from running. Filing it as `BLOCKED` moves it to a pile that carries no obligation.
`FAIL` puts it where an unexecuted assertion belongs.

---

## `BLOCKED` is not `N/A`

They are adjacent and opposite. `N/A` is **declared in the specification** for a behavior
that cannot be driven at all, and is a permanent, reviewed statement. `BLOCKED` is
**computed per run** and means this run could not answer a question that is otherwise
answerable. An evaluator must never infer `N/A`
([`method/12`](../../../method/12-false-greens.md) C3), and must never report either as a
pass.
