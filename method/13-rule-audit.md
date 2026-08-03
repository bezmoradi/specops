# 13 — Rule audit

Every rule in this method, classified by **what actually enforces it**.

This page exists because a live trial produced an uncomfortable result: a corpus applied the
method's rules faithfully and still shipped two defects, both in places where the rule was
*advice*. Advice is enforced by whether the author happened to think of it, which is another
way of saying it is not enforced. The purpose of this audit is to make that visible per rule,
so the ones that matter can be promoted and the ones that cannot be enforced can stop
pretending.

## The three classes

| Class | Definition | What it costs when violated |
|---|---|---|
| **M — Mechanical** | A check exists that fires on violation, with no running system and no model | Nothing. The build fails. |
| **W — Checkable with work** | Mechanically decidable in principle; needs an artifact the project may not have (a schema, an event catalogue, a code manifest, tracing) | The gap is silent until someone builds the input |
| **J — Judgment** | Not decidable from the corpus or the declarations. Depends on the author's intent or on domain knowledge | Enforced only by review, and reliably violated at scale |

**The honest ranking is M > W > J, and the honest goal is to shrink J.** Not to zero — some
rules are irreducibly about intent, and pretending otherwise produces a gate that fires on
correct work. But every rule sitting in J is a rule that will be silently violated by
somebody, and the trial is the evidence.

## The audit

### Core rules — [`03-evidence-rules.md`](03-evidence-rules.md)

| Rule | Class | Enforced by |
|---|---|---|
| R1 — the agent never votes | **M** | Evaluator conformance kit; a verdict not derivable from the bundle is an R1 violation |
| R2 — fail closed | **M** | Conformance cases `003`, `008` |
| R3 — no empty captures, no weaker correlator | **M** | Static gate (unbound `{{VAR}}`); conformance case `004` catches the runtime half |
| R4 — bounded poll, never re-run-until-green | **M** | Budget-class check; retry cap counted in the bundle |
| R5 — altitude that survives refactoring | **M** *(partly)* | Path/line/extension grep is M; "no internal type name" is a **J** heuristic, and is reported rather than blocking |
| R6 — every assertion declares provenance | **M** for presence, **W** for truth | Tag presence is grepped; whether `@intent` *is* intent-derived needs a resolvable citation into a document the gate can fetch |

### Verdicts — [`02-verdicts.md`](02-verdicts.md)

| Rule | Class | Enforced by |
|---|---|---|
| All seven verdicts computed | **M** | Conformance kit |
| `INFRA` requires a recorded probe error | **M** | Case `003` |
| `INDETERMINATE` is a closed list of four | **M** | Case `004` |
| `BLOCKED` computed from `requires`, both directions | **M** | Case `006` |
| `KNOWN-DEVIATION` requires a pre-existing `deviation` block with a citation | **M** for shape, **W** for citation resolution | Static gate |
| Verdicts weakened, never strengthened | **M** | Conformance kit |

### Side effects — [`06-side-effects.md`](06-side-effects.md)

| Rule | Class | Enforced by |
|---|---|---|
| Response and side effect are independent verdicts | **J** | Nothing checks that a scenario asserting a `2xx` also asserts its effect. Candidate for a completeness obligation from the API schema. |
| `hold` for invariants, `within` for appearance | **M** | Static gate (B6 row) |
| **Exactly-once needs receipt + bracket** | **M** for the form, **W** for the obligation | The gate catches an unbracketed positive assertion on a stimulus-produced subject; knowing *which* events are exactly-once needs an event catalogue. **This is H1, and it was J until the trial.** |
| Inverted assertions need a positive control | **M** | Static gate |
| Control must be same-subject or proxy, correctly chosen | **J** | Which kind is right depends on whether the subject may legitimately emit the event — not decidable from the corpus |
| Bracket the control | **M** | Static gate can see whether a trailing control exists |
| Seeded subjects need a seed-visibility control | **M** | Static gate |
| Stale-read handling on survival assertions | **W** | Needs the environment's declared consistency model |
| Correlator choice | **J** | Whether a correlator is genuinely per-execution-unique is domain knowledge |

### Gates and completeness — [`09-gates.md`](09-gates.md)

| Rule | Class | Enforced by |
|---|---|---|
| Structural, contract, safety, altitude, hygiene checks | **M** | The static gate itself |
| Exactly-once obligation generated from the event catalogue | **W** | Needs the catalogue. Absent it, report "could not run" — never silently skip. |
| Boundary probes generated from declared types | **W** | Needs a schema, and the *narrowest* declaration in the path. **This is H2.** |
| Every declared response code is elicited by some scenario | **W** | Needs the code manifest |
| Gate precision measured on a real corpus before shipping a rule | **M** | Run the candidate rule over `examples/` and any adopting corpus; count true positives |

### The rest

| Page | Dominant class | Note |
|---|---|---|
| [`04`](04-spec-altitude.md) spec altitude | **M** / **J** | The refactor test is J; its grep-able proxies are M |
| [`05`](05-provenance.md) provenance | **W** | Everything hinges on citation resolvability |
| [`07`](07-isolation.md) isolation | **M** | Teardown presence, fixture scoping, lane partitioning are all structural |
| [`08`](08-evidence-bundles.md) evidence bundles | **M** | Bundle shape is schema-checkable; conformance case `007` catches the missing-assertion half |
| [`10`](10-environments.md) environments | **M** | Binding resolution and `safety` class are mechanical |
| [`11`](11-economics.md) economics | **J** | Advice about cost. Correctly J — nothing here decides a verdict. |
| [`12`](12-false-greens.md) false greens | mixed | Each entry's *rule* line carries its own class; the catalogue is a diagnosis index, not a gate |

## What the audit says

Counting the rules above: the method is mostly **M**, with a **W** band that is entirely
about *inputs the project may not have* — a schema, an event catalogue, a code manifest,
citations that resolve — and a small **J** residue.

Three conclusions worth acting on:

**1. The W band is the real backlog, and it is not a documentation problem.** Every W rule
becomes M the moment someone writes down a declaration that should exist anyway. "Which of
our events are exactly-once" is not a testing artifact; it is a fact about the system that
nobody had recorded. The gate's honest output when it is missing — *this check could not
run* — is more useful than the check would have been, because it names the real gap.

**2. Both trial defects were in rules that were J at the time.** Exactly-once was J and is
now M+W. Boundary values were not a rule at all and are now W. That is the audit's whole
value proposition, and it is also its warning: the remaining J rules are where the next
escape will be. The two largest are *response/side-effect independence* and *control kind
selection*.

**3. Precision is a property of a rule and must be measured before it ships.** A rule is not
finished when it catches the defect; it is finished when it catches the defect *and leaves
correct scenarios alone*. The exactly-once rule's first draft caught the defect at ~12%
precision, which would have flagged a corpus's best scenarios — and
[`09-gates.md`](09-gates.md) states what happens then: the gate gets switched off, and its
true positives go with it. **A rule that gets disabled has negative value**, because it also
consumed the attention that a narrower rule would have earned.

## Adding a rule

State its class in the same table above. If it is **J**, say why it cannot be M or W — and
if the answer is "nobody has built the input yet," it is **W**, not J, and the input belongs
in the backlog rather than the rule in the advice pile.

A new rule ships with its precision measured on at least one real corpus. Not the corpus it
was written against — a rule always scores well on the corpus that inspired it.
