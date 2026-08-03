# Review request — SpecOps

> **Temporary file. Delete before publishing the repository.**
>
> **Status: this review has been completed and its findings applied.** Four blockers, twelve
> majors, and the nits were all verified against the files and fixed — the grammar gained
> `setup`/`control` blocks and `hold` semantics, the store-observation-versus-escape-hatch
> contradiction was resolved, the FC2/OT1/OT3/OC3 scenarios were rewritten, and six new
> catalogue entries were added (A4, B5, B6, B7, C5, G5, G6). A threat model, a conformance
> kit, a one-page summary, and a worked evidence bundle were added in response.
>
> Two review recommendations were **declined**, with reasons: reorganizing the repo around
> the evidence bundle rather than the specification (the adoption unit is the spec, and the
> human-readability property lives there), and the "key shapes" framing of the store-evidence
> contradiction (the citation was imprecise; the real issue was the persistence-refactor
> tension, which is now carved out explicitly in `method/04`).
>
> Counts below were accurate at the time of the request. The catalogue now has **32**
> entries, not 21 — the stale number was itself a small instance of the drift the method
> warns about, and the reviewer caught it.

You are being asked to review a repository that contains no code. That is deliberate, and
it is the first thing to have an opinion about.

---

## 0 — What I need from you, in one paragraph

This repo proposes a **method** for writing system specifications that an AI agent
executes against a running deployment, and whose verdicts a human can trust. I believe the
method is right. I am much less sure the *repository* is right — that it is honest about
its limits, that it is actionable rather than merely principled, that it has not
over-engineered a taxonomy nobody will use, and that its central claim survives contact
with someone who did not write it. **I want you to try to break the argument, not polish
the prose.** A review that finds one load-bearing flaw is worth more to me than a hundred
line edits.

---

## 1 — What this repository is

SpecOps is a **methodology and reference baseline**, not a tool. Nothing to install. You
copy templates into your own repo and use the prompts to get your coding agent to write
and later execute behavioral specifications for your own system.

The premise, in short:

- The usual AI workflow has an agent write both the implementation and the tests. Both can
  encode the **same misunderstanding** of the requirement, and then agree with each other
  perfectly. A green suite says nothing.
- Separately: when an agent both *executes* a check and *decides whether it passed*, it
  rationalizes. Shown a `201 Created` and no trace of the downstream email, it concludes
  the email probably got sent. This is the dominant failure mode, not an occasional lapse.

So the method makes two cuts:

```
specification / implementation / verification      — three artifacts, not one
acquire evidence / judge evidence                  — agent does the first, rules do the second
```

The agent keeps the job it is genuinely good at — working out *how* to observe something
across an API, a queue, a log, a datastore, a metric. It loses the job it is bad at —
deciding whether what it found is good enough.

## 2 — Where this came from

This is not a design exercise. It was extracted from a working suite:

| | |
|---|---|
| Specifications | 150 files across 3 services |
| Scenarios | 722 |
| Specification prose | ~29,500 lines |
| Runner code | ~4,250 lines of shell |
| Environment | live multi-service cloud deployment, run post-deploy |

Its strongest empirical result: used as the regression gate for a full internal
restructuring of one service (every package moved, no observable behavior changed), it
returned **563 PASS / 0 FAIL**, with all 21 non-passing scenarios being pre-documented
fixture gaps rather than discovered regressions. A second service's suite returned
**53 PASS / 0 FAIL** through the same refactor. That is the "specifications must survive
refactoring" claim, tested rather than asserted.

**It was also audited, and it failed in five specific ways.** Those five failures are why
`method/` has twelve pages instead of seven:

| Page | Gap it closes | Present in the source suite? |
|---|---|---|
| `05-provenance` | Specs written by verifying against the implementation are a regression net, not an independent oracle | **No** |
| `08-evidence-bundles` | No machine-readable evidence archive; a past PASS is unauditable | **No** |
| `09-gates` | No pre-merge gate; the only confidence gate is a multi-hour post-deploy run | **No** |
| `10-environments` | Regions, table names, cluster commands inlined in specs; bound to one deployment | **No** |
| `11-economics` | Agent in the loop on every run; slow, costly, non-reproducible | **No** |

The other seven pages distill what the suite got **right**: fail-closed verdicts, the
no-weaker-correlator rule, bounded polling, correlator-first evidence ranking, the
positive-control recipe for "this must not happen" scenarios, rate-limit-aware lane
partitioning, and the altitude rule.

So the repo is deliberately part success story, part post-mortem. Judge whether that
framing is honest or whether it is having it both ways.

## 3 — Inventory

30 files, ~28,700 words.

| Path | Words | Role |
|---|---|---|
| `README.md` | 1,185 | Positioning, philosophy, adoption path |
| `method/01-overview.md` | 1,232 | Separations, classification, **scope boundary** |
| `method/02-verdicts.md` | 1,029 | Six-verdict algebra |
| `method/03-evidence-rules.md` | 1,589 | The six load-bearing rules + assertion grammar |
| `method/04-spec-altitude.md` | 1,096 | The refactor test; what specs must never name |
| `method/05-provenance.md` | 1,172 | `@intent` vs `@contract` |
| `method/06-side-effects.md` | 1,495 | Correlators, evidence ranking, inverted verdicts |
| `method/07-isolation.md` | 1,072 | Lanes, fixtures, lifecycle |
| `method/08-evidence-bundles.md` | 967 | Auditable verdicts |
| `method/09-gates.md` | 989 | Static gate vs verification run; selection |
| `method/10-environments.md` | 848 | Logical→physical bindings; safety classes |
| `method/11-economics.md` | 892 | Plan caching; cost tiering |
| `method/12-false-greens.md` | 2,392 | **The catalogue — 21 traps** |
| `templates/` | 2,284 | spec · guide · environment.yaml · index · RUNLOG |
| `prompts/` | 2,994 | author · execute · review |
| `examples/` | 2,825 | Filled-in guide + 3 worked specs |
| `adoption/` | 2,008 | Getting started · checklist |
| `CONTRIBUTING.md`, `CLAUDE.md` | 1,384 | Contribution bar; agent guidance |

## 4 — Decisions made, and what was rejected

Argue with any of these. I have listed the alternative I discarded so you can tell me I
discarded the right one for the wrong reason, or the wrong one entirely.

| Decision | Alternative rejected | Why |
|---|---|---|
| Markdown with **typed fenced blocks**; prose explicitly non-normative | Pure YAML/JSON spec format | Prose is why a human reviews these at all. But prose cannot be load-bearing, hence the typed blocks. The rule is: *anything that decides a verdict lives in a typed block.* |
| Six verdicts | The usual two, or three | `INFRA` separates "environment broke" from "system broke" so retries are legitimate *and counted*. `BLOCKED` computed from declared preconditions stops it being the drain failures escape through. |
| **No implementation in this repo** | Ship a CLI alongside | A method that ships with a tool gets read as documentation for the tool, and dies with it. Also: I do not want the format's credibility to depend on my runner being good. |
| Numbered, citable `method/` pages | Topic-named pages | So people can cite `method/03-evidence-rules.md#R3` in their own code reviews. Adoption happens through citation. |
| Fictional worked examples (an orders service) | Real examples from the source system | Portability and no leakage. Cost: less visceral. |
| `MIT` | `Apache-2.0`, or `CC-BY` for prose | Chosen for simplicity. Genuinely unsure — see open questions. |

## 5 — The open questions I already have

These are my own doubts. I am listing them so you do not spend the review rediscovering
them — **and so you can tell me which ones I am wrong to be worried about, and which ones
are worse than I think.**

**Q1 — Does the repo ship advice it makes impossible to follow?**
The central rule is *the agent never votes* — verdicts are computed from recorded evidence
by a deterministic evaluator. **There is no evaluator.** Every adopter's agent will, in
fact, vote. Is the method therefore aspirational in exactly the place it claims to be
rigorous? Should the repo ship a minimal reference evaluator despite the "no
implementation" decision, or does stating the rule without enforcement still do useful
work?

**Q2 — Does plan caching destroy the thing that justified the agent?**
`11-economics` proposes caching the resolved plan after the first success and replaying it
without a model. But once you have a frozen plan, you have a scripted test — and scripted
tests are what the agent was supposed to improve on. Is this a genuine synthesis, or am I
smuggling the old brittleness back in and calling it an optimization?

**Q3 — Is `@intent` / `@contract` a real practice or bureaucratic theater?**
Provenance tagging is elegant on paper. In practice, will anyone do it honestly after
week three, or will everything get tagged `@contract` mechanically until the ratio means
nothing? Is there a version of this that is self-enforcing rather than self-reported?

**Q4 — Are six verdicts one or two too many?**
I believe `INFRA` and computed `BLOCKED` pay for themselves. I am less sure about
`INDETERMINATE` — it may be a distinction without a difference from `FAIL` in practice.
Where is the line between a taxonomy that captures real signal and one that teams will
collapse back to pass/fail within a month?

**Q5 — Is 28,700 words the wrong size?**
*The Twelve-Factor App* is roughly 4,000 words and changed how an industry deploys
software. This is seven times longer. Is the length earned by the domain's genuine
complexity, or is this a book pretending to be a methodology? If it should be shorter,
what is the 4,000-word version — and what gets cut?

**Q6 — Is the scope boundary honest, or a hedge?**
`01-overview` states plainly that this cannot find races, performance regressions,
input-space gaps, or algorithmic errors. Is that admirable honesty, or does it carve out so
much that the remaining claim is smaller than the repo's tone implies?

**Q7 — Is the classification right, and is it undersold?**
The repo classifies itself as "agent-executed, grey-box, end-to-end regression testing with
cross-system evidence" and explicitly *refuses* the label "AI-native autonomous QA." I
think that honesty is correct positioning. It may also be why nobody reads it. Where is the
line between accurate and unmarketable?

**Q8 — Is the false-greens catalogue complete?**
`12-false-greens.md` has 21 entries. I am certain it is missing some. What is missing is
worth more than anything else you could tell me.

**Q9 — The name.**
"SpecOps" collides with Specops Software (an Outpost24 company) whose registered trademark
explicitly covers *"software for user identification, authentication, and verification."*
The npm package `specops` has ~6,900 downloads/month. I chose the name anyway. Tell me
honestly whether that is a survivable decision for an open-source methodology or a
foreseeable problem.

**Q10 — Is there a Monday morning?**
`adoption/getting-started.md` claims a path from zero to first trustworthy green in half a
day. Walk it as a skeptical newcomer. Does it actually work, or does it assume context the
reader does not have?

## 6 — What I want you to attack

Please spend most of your effort here. In priority order:

### 6.1 — Attack the premise

Not the wording — the argument.

- Is "specification as independent oracle" a real property, or a comforting story? The repo
  concedes that most specs get written by reading the implementation. Does `05-provenance`
  actually rescue the claim, or just make its failure legible?
- Is the acquire/judge split achievable in practice? Construct the case where acquisition
  and judgement cannot be cleanly separated — where deciding *what to acquire* already
  presupposes the answer. If that case is common, the whole architecture bends.
- The repo asserts that agents rationalize toward green. Is that a durable property of
  these systems, or an artifact of current models that a better one erases in a year? If
  the latter, half of `method/` is scaffolding around a temporary problem.

### 6.2 — Find the false greens in the false-greens catalogue

The catalogue is the repo's flagship claim to originality. Read `12-false-greens.md` as an
adversary:

- Which entries are real, and which are folklore dressed as mechanism?
- Which entries' stated **rule** does not actually close the stated **mechanism**?
- What traps are missing? Think about categories the repo does not have: time and clock
  skew, caching layers, retries in the system under test, partial failures, schema
  evolution, multi-region, feature flags, data that expires mid-run.
- Are there false greens *created by this method's own rules*? A rule that produces a new
  blind spot while closing an old one is the most valuable finding available here.

### 6.3 — Break the examples

`examples/` contains three worked specifications and a filled-in execution contract. They
are load-bearing: they are what people will copy.

- For each scenario, construct the broken system that still passes it. If you find one,
  that is a top-priority finding — the repo is teaching a hole.
- Is the negative-space example (`orders-tenancy.md`) actually correct? It is the hardest
  one, and it makes strong claims about existence oracles and positive controls.
- Do the examples violate any of the repo's own rules?

### 6.4 — Test internal consistency

Twelve normative pages cross-referencing each other is a lot of surface for contradiction.

- Does any page contradict another? Specifically check: `02-verdicts` vs `03-evidence-rules`
  on retry semantics; `09-gates` vs `11-economics` on what may gate what; `06-side-effects`
  vs `10-environments` on where budgets live.
- Does `templates/spec.md` actually conform to the grammar in `03-evidence-rules`?
- Do the prompts in `prompts/` ask for anything the method forbids, or forbid anything it
  requires?

### 6.5 — Judge it as a product

- If you found this repo on GitHub, would you adopt it? What is the single thing that would
  most change that answer?
- What is the first thing a hostile Hacker News commenter says, and is it right?
- What is missing entirely? A page, a section, an artifact, a diagram, an argument.

## 7 — Suggested review order

1. `README.md` — 10 minutes. Judge the pitch cold.
2. `method/01-overview.md`, `02-verdicts.md`, `03-evidence-rules.md` — the core argument.
3. `method/12-false-greens.md` — the flagship. Read adversarially.
4. `examples/` — all four files. Try to break every scenario.
5. `method/05` and `08`–`11` — the five pages fixing the source suite's gaps. These are the
   least battle-tested and most likely to be wrong.
6. `templates/` and `prompts/` — check conformance and practicality.
7. `adoption/getting-started.md` — walk it as a newcomer.
8. Everything else, if you still have appetite.

## 8 — What not to spend time on

- **Prose polish, typos, comma placement.** Note them in a single bucket and move on.
- **Formatting and table styling.**
- **The absence of `CODE_OF_CONDUCT.md` and `SECURITY.md`.** Known; they are coming.
- **Suggesting the repo should ship a CLI**, unless your argument is specifically the one in
  Q1 (that the no-implementation stance makes the central rule unfollowable). That version
  of the argument is very welcome.
- **Agreeing with me.** If a section is fine, say "fine" in three words and spend the time
  elsewhere.

## 9 — The bar

I want this to be the best thing of its kind, and I think that is achievable because the
field is genuinely thin. The adjacent work is either **UI-first, closed, and vendor-owned**
(the current crop of agentic QA platforms), **deterministic but blind past the HTTP
response** (Tavern, Step CI, Hurl, Karate, REST Assured), **dependent on hand-written glue**
(Cucumber, Concordion, Robot Framework), or **about generating code rather than verifying
deployments** (Spec Kit, Tessl, Kiro, and the spec-driven-development toolchain generally).

Nobody I could find has published an open method for **agent-executed, evidence-based,
multi-system verification with explicit anti-false-green controls.** That is the gap this
aims at. Its unfair advantage is not the Markdown format — it is the catalogue of ways an
agent-executed suite lies to you, which was learned expensively.

So the standard I am asking you to hold it to is not "is this good documentation." It is:

> **Would a competent engineer, having read this, build a verification suite that is
> meaningfully harder to fool than the one they would have built otherwise?**

If the answer is no, tell me exactly where the method fails to deliver that, and I would
rather hear it now than after publishing.

## 10 — Output format

Please structure findings as:

```
[SEVERITY] <file>:<section> — <one-line claim>

Mechanism:  why this is wrong, concretely
Consequence: what it costs someone who follows it
Fix:        a specific rewrite, not "consider revising"
```

Severities:

- **Blocker** — a claim that is wrong, or guidance that would make a reader's suite worse.
- **Major** — a load-bearing gap, contradiction, or unsupported assertion.
- **Minor** — a real improvement that is not load-bearing.
- **Nit** — bucket these together, one line each.

Then close with three things:

1. **The single most valuable change**, if I only make one.
2. **Your answers to the ten open questions in §5**, however brief. Where you disagree with
   my framing of a question, say so — a badly framed question is itself a finding.
3. **What you would have written instead**, if you disagree with the whole approach. That
   is the most useful thing you can give me, and I would rather have it as a paragraph of
   disagreement than as a page of agreement.
