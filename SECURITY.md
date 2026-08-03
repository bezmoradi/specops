# Security

## There is no software here

This repository ships **no executable code**: no package, no binary, no CLI, no
dependencies. Nothing here runs, so there is nothing here to exploit in the usual sense, and
no dependency tree to advisory-scan.

That does not make the security surface zero. It moves it.

## The actual surface

This repository publishes **instructions that people give to AI agents with production
credentials**. That is the risk, and it is worth being explicit about it.

An adopter hands their agent [`prompts/execute-specs.md`](prompts/execute-specs.md), an
environment binding, and access to a deployed system. The agent then sends requests, reads
stores, publishes events, and — in some scenarios — writes state through escape hatches. The
guardrails that keep this from being reckless are all *textual*, and they live in this
repository:

| Guardrail | Where | What it prevents |
|---|---|---|
| The `safety` class (`mutable` / `read-only` / `forbidden`) | [`method/10-environments.md`](method/10-environments.md), [`templates/environment.yaml`](templates/environment.yaml) | A mutating scenario running against production |
| Escape hatches restricted to `requires` and `teardown`, and to `mutable` environments only | [`method/04-spec-altitude.md`](method/04-spec-altitude.md) | Arbitrary state writes normalized as a testing technique |
| Executor privilege — least privilege, and the signing-key warning | [`method/10-environments.md`](method/10-environments.md) | A suite that requires credentials able to forge identity |
| Redaction at capture, not at display | [`method/08-evidence-bundles.md`](method/08-evidence-bundles.md), [`prompts/execute-specs.md`](prompts/execute-specs.md) | Credentials and personal data persisted into evidence bundles and committed |

**A defect in any of those is a security defect in this project**, even though no code is
involved. Examples of things worth reporting:

- A prompt, template, or example that would lead an agent to mutate a `read-only` or
  `forbidden` environment.
- An example that captures a credential, token, or personal data into a bundle without
  redaction — bundles get committed and attached to tickets.
- Guidance that requires broader privilege than the scenario needs (the signing-key note in
  `examples/negative-space/orders-tenancy.md` OT4 is the pattern to follow: state the
  privilege and warn).
- Any instruction that could be read as authorizing an agent to widen its own access,
  disable a guardrail, or work around a `safety` class rather than stopping and reporting.
- A phrasing an adopter's agent could plausibly misread in the dangerous direction. This
  counts even if the correct reading is available — the prompts are executed by models, not
  parsed by compilers, and "a careful reader would understand" is not a control.

## What is explicitly out of scope

- Vulnerabilities in *your* system found by running these specifications. Report those to
  whoever owns that system.
- Vulnerabilities in an agent, model, runner, or CI platform you use to execute them.
- Prompt injection against your own agent through your own data. Real, but it is a property
  of your harness — see the [threat model](method/01-overview.md#threat-model), which states
  plainly what this method does and does not defend against. A **fabricated evidence bundle
  is not defended against**; that is documented, not a finding.

## Reporting

Use GitHub's [private vulnerability
reporting](https://github.com/bezmoradi/specops/security/advisories/new) for anything you
would rather not open in public. Otherwise, open a normal issue — most findings here are
better discussed in the open, because the fix is usually a clarification everyone benefits
from reading.

Please include the file and line, and the concrete misuse it enables: which environment gets
mutated, which credential leaks, which guardrail is bypassed. A specific scenario is worth
more than a severity rating.

There is a single maintainer and no service-level commitment. Expect acknowledgement rather
than immediacy.

## No advisories, by construction

Because nothing is published as a package, there is no version to pin and no advisory to
consume. Fixes land as commits. If you have vendored any of this into your own repository —
which is the intended use — you are the one who has to re-sync, and
[`method/09-gates.md`](method/09-gates.md) is the argument for gating that statically rather
than remembering to.
