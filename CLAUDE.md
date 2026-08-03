# CLAUDE.md — working in the SpecOps repository

Guidance for Claude Code and other coding agents editing **this** repo.

Note the recursion: SpecOps is a method for instructing agents, maintained with the help
of agents. If a rule here is annoying to follow, that is data about the method.

## What this repository is

A **methodology**. Prose, templates, prompts, and worked examples. There is no build, no
test suite, no dependency tree, and no CLI.

**Do not add an implementation.** No `cli/`, no `package.json`, no `pyproject.toml`, no
runner. If SpecOps ever grows a tool, it lives in a separate repository. Keeping this one
implementation-free is deliberate: a method that ships with a tool gets read as
documentation for the tool, and then dies with it.

**`conformance/` is not an exception to that.** It contains a normative description of
evaluator behavior plus golden `(spec, bundle, expected verdict)` triples as **data**. No
code executes them here. That is what lets a second implementation exist, and it is the
answer to "the method forbids the agent from voting but ships nothing that stops it" — it
does not stop it, it makes an adopter able to audit their own executor.

## Layout, and what each part is for

| Path | Status | Rule |
|---|---|---|
| `method/` | **Normative** | Numbered and citable. People link `method/03-evidence-rules.md#no-weaker-correlator` from their own code reviews. Do not renumber without leaving a deprecation note. |
| `templates/` | Copied by users | Must stay generic. A template that only works for one stack is an example, not a template. |
| `prompts/` | Operational | Written to be pasted at an agent verbatim. Test a change by actually running it. |
| `examples/` | Illustrative | Concrete and complete. May name a stack; must still obey the altitude rule. |
| `adoption/` | Onboarding | May be opinionated and may name vendors. |

## Writing rules

1. **Rules are imperative and testable.** "Prefer" and "consider" are not rules. If it can
   be violated with nobody noticing, write MUST and state what noticing looks like.
2. **Every normative claim states its failure mode.** A rule earns its place by naming
   what breaks without it, and whether that break is loud or silent. Silent breaks are the
   whole point of this repo — they get priority and they get detail.
3. **One rule, one home.** Cross-link; never restate. Duplicated guidance drifts, and the
   reader who finds the stale copy has no way to know it is stale.
4. **No vendor names in `method/`.** No cloud provider, no log aggregator, no model
   vendor, no orchestrator. Those belong in `examples/` and `adoption/`. A normative page
   that names a vendor stops being portable and starts being a tutorial.
5. **Examples must obey the altitude rule.** No handler names, class names, file paths, or
   line numbers in any example specification — that is the rule the method teaches
   (`method/04-spec-altitude.md`), and an example that breaks it teaches the opposite.
6. **Dense over long.** This repo is read by people deciding whether to adopt, and loaded
   as context by agents. Padding costs both.

## The README

`README.md` is the entry point and the positioning document. Do not rewrite,
restructure, or "refresh" it as a side effect of another task. Change it when the change
is the task, and when a human asked for that specific change.

This mirrors the discipline the method itself recommends for the documents a project's
credibility rests on: the value of the front door comes from a human having verified it.

## Cross-repo note

This method was extracted from a working behavioral-test suite in a separate,
private multi-service project. The extraction runs **one way**: lessons flow from that
suite into `method/`, generalized and stripped of anything specific to it. Do not import
its infrastructure details — regions, table names, cluster commands, host names, service
names — into this repository. If a rule can only be stated using those details, it is not
yet a rule; it is a local practice.

The reverse direction — applying SpecOps back to that suite — is a separate task in that
repository, not this one.

## Consistency checks before finishing

There is no automated gate, so these are manual. **The repo must be self-conformant** — it
teaches a static gate, so its own corpus has to pass one:

- **Grammar conformance.** Does every typed block in `templates/` and `examples/` use a block
  type and an operator defined in `method/03-evidence-rules.md`? An example that reaches
  outside the published grammar is teaching agents to improvise, which the method identifies
  as where lying starts.
- **No verdict-deciding prose.** Does any example put a capture, an assertion, or a control
  outside a typed block? That violates the method's own central rule.
- **Every `{{VAR}}` is bound** in a typed block before use, in every template and example.
- **`within` versus `hold`.** Does any assertion shaped `count ==` / `still` / `unchanged`
  use `within` **on a subject that predates the stimulus**? That is the B6 false green, in
  the artifacts people copy. The qualifier matters in both directions: a `count ==` receipt
  for something the stimulus creates is a legitimate `within`, and flagging it would reject
  `examples/async-event/order-cancelled.md` FC2, which is correct as written.
- **Conformance cases.** Does every `expected.json` verdict follow from `EVALUATOR.md`'s
  ordering, and does every reason and finding code it uses appear in that page's
  vocabulary tables? The cases are the executable half of the method; a case that disagrees
  with the normative page makes both unusable.
- **Every inverted (`absent through`) assertion has a `control` block**, of the correct kind
  (same-subject or proxy), and states its residual gap if proxy.
- **Every seeded-subject negative scenario has a seed-visibility control.**
- Do the page counts and entry counts stated in `README.md` and `method/00-summary.md` match
  reality? (The catalogue count has been wrong before.)
- Does every cross-referenced `method/` page and anchor exist?
- Did a new rule get added in more than one place?
