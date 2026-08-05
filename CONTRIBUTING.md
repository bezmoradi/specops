# Contributing to SpecOps

SpecOps is a **methodology**, not a program. There is no build, no test runner, and
no dependency tree. Contributions are prose, templates, prompts, and worked examples.

That makes the bar different from a normal repo: a bad paragraph here does more damage
than a bad function would, because it propagates into other people's suites and produces
suites that report green while their systems are broken. Please read this before opening
a PR.

## What is most valuable

In rough order:

1. **A new entry in [`method/12-false-greens.md`](method/12-false-greens.md).** A concrete
   way an agent-executed suite reported PASS while the system was broken. This is the
   highest-value contribution in the repo. See the entry format below.
2. **A worked example** covering a surface not yet in `examples/` — gRPC, GraphQL,
   WebSocket, cron/scheduled jobs, batch pipelines, mobile backends.
3. **A prompt refinement** in `prompts/` that measurably improved spec quality or verdict
   fidelity with your agent. Say which agent and what changed.
4. **A conformance case** in [`conformance/cases/`](conformance/cases/). A case is a
   directory holding `spec.md`, `bundle.json`, and `expected.json`, plus one sentence naming
   the mistake it catches. The most valuable encode a catalogue entry from `method/12`, since
   that is how a written warning becomes something an evaluator can be *tested* against.
   Read an existing case first — [`008`](conformance/cases/008-hold-missing-final-sample/spec.md)
   is the clearest shape — and make sure every reason and finding code you use appears in
   [`conformance/EVALUATOR.md`](conformance/EVALUATOR.md). A case whose expected verdict does
   not follow from that page is a change to the page, and should be argued as one.
5. **Corrections.** If a rule here is wrong, or right for the wrong reason, say so. The
   rules are empirical, not axiomatic.

## What is not wanted

- **A CLI, a runner, or a package tree.** If SpecOps ever grows an implementation it will
  live in a separate repo. Keeping this one implementation-free is deliberate: the method
  has to survive any particular tool, and a tool in the repo makes the method look like
  documentation for the tool.
- **Vendor-specific instructions in `method/`.** Cloud providers, log aggregators, and
  model vendors belong in `examples/` and `adoption/`, never in the normative pages.
- **Rules with no evidence.** "This seems safer" is not a reason. See below.

## The evidence bar

Every normative claim in `method/` must be traceable to something that actually happened.
A rule earns its place by having caught a real defect, or by having failed to and thereby
teaching what the rule should have been.

When proposing a rule, state:

- **The failure it prevents.** Concretely: what was broken, what the suite reported.
- **How it fails without the rule.** Loudly or silently. Silent failures get priority.
- **The cost of following it.** Every rule has one. Name it honestly.

A rule that cannot state its failure mode is a preference, and preferences belong in
`adoption/`, not `method/`.

## False-green entry format

```markdown
### <short imperative title>

**Symptom.** The suite reported PASS. The system was broken in this way: …

**Mechanism.** Why the assertion passed anyway: …

**Rule.** <the one-line rule that closes it>

**Cost.** What following the rule costs you: …
```

Keep the mechanism section precise. "The agent was confused" is not a mechanism; "the
correlator was empty and the agent fell back to matching on the message string, which
matched a sibling scenario's log line" is.

## Writing conventions

- **`method/` is normative. Everything else is not.** Pages in `method/` are numbered and
  stable so people can cite them (`method/03-evidence-rules.md#no-weaker-correlator`) in
  their own code reviews. Do not renumber without a deprecation note.
- **Rules are imperative and testable.** "Prefer" and "consider" are not rules. If it can
  be violated without anyone noticing, write it as MUST and say what noticing looks like.
- **One rule, one home.** Cross-link rather than restate. Two copies of a rule drift, and
  a reader who finds the stale one has no way to know.
- **No internal symbols in examples.** Examples demonstrate specs, and specs must never
  name handlers, classes, files, or line numbers — that is the altitude rule
  ([`method/04-spec-altitude.md`](method/04-spec-altitude.md)). An example that violates it
  teaches the wrong thing.
- **Keep prose dense.** This repo is read by people deciding whether to adopt a method and
  by agents loading it as context. Both are hurt by padding.

## Pull requests

- One concern per PR. A false-green entry and a template change are two PRs.
- Changes to `method/` need the evidence bar above in the PR description.
- Changes to `templates/` should say which example you re-ran to confirm the template
  still produces a valid spec.

## Tone

Be straightforward and assume competence. Disagreement about a rule is the point of the
repo; make the argument, cite the failure, and let the evidence settle it.

## Security

If a change to a prompt, template, or example could lead someone's agent to mutate a
protected environment or capture a credential into a bundle, that is a security issue rather
than a style issue — see [`SECURITY.md`](SECURITY.md). Nothing here executes, but everything
here is executed *by* something with credentials.
