# Getting started

Day one to first trustworthy green. Budget half a day for the first scenario and an hour
each for the next ten.

**Do not start by writing a hundred specifications.** Start with one, run it, and read the
evidence. Almost everything that goes wrong with this method goes wrong in the gap between
"the run said PASS" and "the system actually does that," and you find that gap by reading
one bundle carefully, not by scaling.

---

## Step 0 — Read one page, then two more

[**`method/00-summary.md`**](../method/00-summary.md) — the whole method, five minutes.

Then [`method/02-verdicts.md`](../method/02-verdicts.md) and
[`method/03-evidence-rules.md`](../method/03-evidence-rules.md) for the grammar and the
verdict algebra. About twenty minutes total. The rest waits until you hit the situation it
describes.

## Step 1 — Set up the folder

```
your-repo/
├── specs/
│   ├── guide.md              ← copy templates/guide.md, fill it in
│   ├── index.md              ← generated later
│   └── orders-create.md      ← your first specification
├── environments/
│   └── staging.yaml          ← copy templates/environment.yaml
├── fixtures/
├── runs/                     ← gitignored
└── RUNLOG.md                 ← copy templates/RUNLOG.md
```

Add to `.gitignore`:

```
.env
.run/
runs/
```

## Step 2 — Fill in one environment

Copy [`templates/environment.yaml`](../templates/environment.yaml). Set `safety: mutable`
only if this environment genuinely may be mutated. If you are pointing at production,
`safety: read-only`, and accept the smaller scenario set.

The section people skip is `evidence`. Answer three questions now, because you will need
them at 2am later:

1. **How do I read this service's logs, in this environment, right now?**
2. **How many replicas does it run?** (If more than one: a request's evidence exists on
   exactly one of them, so your log read must aggregate.)
3. **Can I read the datastore directly, and should I?**

## Step 3 — Fill in the execution contract

Copy [`templates/guide.md`](../templates/guide.md) to `specs/guide.md`.

The section that matters most is **correlators**. Answer:

> When my API handles a request and something happens downstream, what identifier appears
> in **both** places?

Common answers: a propagated trace identifier, or an identifier your API returns and the
downstream evidence contains.

If the answer is "nothing" — you have found something more valuable than a test. There is
currently no way to prove any downstream effect was caused by any particular request. Fix
that first; it is worth more than the entire suite you were about to write.

## Step 4 — Write one specification

Pick an endpoint with a **downstream side effect**. Not your simplest endpoint — the
simplest ones teach you nothing, because their response is the whole story.

Give your agent [`prompts/author-specs.md`](../prompts/author-specs.md) plus the endpoint.

Then review what comes back against
[`adoption/checklist.md`](checklist.md). Expect to send the first one back. The two
questions that catch the most:

- *"Where did this number come from?"* for every status and threshold.
- *"Would this still pass if I renamed every class?"*

## Step 5 — Run it

Give your agent [`prompts/execute-specs.md`](../prompts/execute-specs.md), the
specification, and the environment name.

## Step 6 — Read the evidence bundle, not the verdict

**This is the step that determines whether any of this works.**

A worked one is at [`examples/bundle/orders-create-OC1.json`](../examples/bundle/orders-create-OC1.json)
— read it first so you know what a good bundle looks like before you judge your own.

Open yours and check:

- [ ] Does **every** assertion appear, including the ones that passed? A missing one was
      never evaluated.
- [ ] Does the side-effect assertion have an actual **probe result** behind it — a real
      query and a real return value?
- [ ] Is the correlator the one your specification declared, or something broader?
- [ ] Are the probe invocations recorded verbatim?
- [ ] How many attempts did each polled step take, and how close to its budget?

Then do the decisive test: **break the side effect on purpose** — stop the consumer,
misconfigure the topic, whatever is cheap — and re-run.

If the scenario still passes, your side-effect assertion is decorative, and you have just
learned that before building a hundred more like it. This single check is worth more than
any amount of reading.

## Step 7 — Write ten more, then stop and review

Cover one area properly rather than every area shallowly. Include:

- at least one **unauthenticated** scenario
- at least one **cross-tenant** scenario, with a **seeded** foreign resource
- at least one **asynchronous** side effect
- at least one scenario where something must **not** happen, with a positive control

Then run [`prompts/review-specs.md`](../prompts/review-specs.md) over all ten with a
different agent session. Fix what it finds. The patterns it surfaces will repeat across
everything you write afterward, which is why ten is the right number to stop at.

## Step 8 — Add the static gate

Once you have twenty or so specifications, drift starts. Add the pre-merge checks from
[`method/09-gates.md`](../method/09-gates.md) — start with the five cheapest:

1. Every specification parses.
2. Every `{{VAR}}` is defined in a typed block before use.
3. Every mutating scenario has a teardown.
4. Every referenced scenario id exists.
5. **No `count ==` / `still` / `unchanged` assertion uses `within` on a subject that predates
   the stimulus** — it must be `hold`. Mechanically: every `store(…)` count, plus any stream
   count not pinned to a correlator minted in that scenario. This one is five lines of grep
   and it catches the highest-yield false green in the catalogue. Do not drop the qualifier:
   the same shape used as a *receipt* for something the stimulus creates is a legitimate
   `within`, and a gate that flags those gets switched off.

These need no running system and no model. They belong on every pull request.

## Step 8b — Audit your evaluator

Whatever is computing your verdicts — a runner you wrote, or the agent following the execute
prompt — run it against [`conformance/`](../conformance/) and confirm it agrees. The case
that matters most is `002`: `within` and `hold` must produce **different** verdicts on the
same bundle. An evaluator that treats them alike will pass every idempotency scenario you
ever write.

## Step 9 — Write the first run log entry

Copy [`templates/RUNLOG.md`](../templates/RUNLOG.md). Record the counts, the build, and —
most importantly — **what you did not run and why.**

Six months from now, during an incident, this is the file someone opens.

---

## Common early mistakes

| Mistake | What happens | Fix |
|---|---|---|
| Starting with a hundred specifications | A hundred specifications with the same flaw, discovered later | One, then ten, then review |
| Trusting the first green | The side-effect assertion was never really evaluated | Break the side effect on purpose and re-run |
| Correlating by time window | Passes alone, collides under parallelism, passes on retry | Mint or propagate an identifier |
| Skipping teardown "just for now" | The next run's fixtures are polluted; failures become unattributable | Teardown from the first specification |
| Asserting message copy | Fails on a copy edit, then gets weakened until it fails on nothing | Assert stable codes |
| Retrying a red | A real regression ships | Bounded polls; `INFRA` for environment errors only |
| Letting the agent report the verdict | The suite becomes incapable of failing | Verdicts computed from recorded evidence |

## Applying this to an existing suite

If you already have a behavioral suite, do not rewrite it. Apply this method as an audit,
in this order — cheapest and highest-yield first:

1. **Add the static gate.** Cheap, immediate, catches existing drift on day one.
2. **Audit for false greens** using [`method/12-false-greens.md`](../method/12-false-greens.md).
   Start with the scenarios you would least like to be wrong about.
3. **Tag provenance** on those same high-value scenarios. Publish the ratio, whatever it is.
4. **Start archiving evidence bundles.** No back-fill; from now on is enough.
5. **Extract environment bindings** from whatever is currently inlined in the
   specifications.
6. **Add plan caching** last, once the rest is stable. It is an optimization, and
   optimizing an unreliable suite makes it unreliable faster.
