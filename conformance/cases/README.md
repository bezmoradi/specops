# Golden cases

Each case is a directory containing `spec.md`, `bundle.json`, and `expected.json`. Run your
evaluator over the spec/bundle pair and compare to `expected.json`.

Most cases carry several **fragments** — `fragment_A`, `fragment_B`, … — that share one
bundle. The fragments are the point: they are usually near-identical inputs that must
produce **different** outputs, and an evaluator's bug is far more often "returns the same
verdict for both" than "returns the wrong verdict for one."

## Status

| Case | Written | Catches |
|---|---|---|
| `001-within-appearance` | ✅ | `within` is an appearance check — the over-correction from 002 |
| `002-hold-invariant` | ✅ | Appearance semantics applied to an invariant — [`12`](../../method/12-false-greens.md) B6 |
| `003-empty-vs-errored` | ✅ | `FAIL` versus `INFRA` classification — [`12`](../../method/12-false-greens.md) C5 |
| `004-empty-capture` | ✅ | Unresolved correlator → `FAIL`, not a fallback — [`12`](../../method/12-false-greens.md) B1 |
| `005-absent-through-no-control` | ✅ | Inverted verdict without a live control — [`12`](../../method/12-false-greens.md) G1 |
| `006-blocked-computed` | ✅ | `BLOCKED` computed, not judged — [`12`](../../method/12-false-greens.md) C2 |
| `007-no-short-circuit` | ✅ | A missing assertion is not a passing one — [`12`](../../method/12-false-greens.md) A2 |
| `008-hold-missing-final-sample` | ✅ | `hold` with no sample at expiry |
| `009-exactly-once` | ✅ | An exactly-once obligation satisfied by "at least one" — [`12`](../../method/12-false-greens.md) H1 |
| `010-known-deviation` | ✅ | A known defect reported as `PASS`, and a *fixed* defect reported as `FAIL` |

## Where to start

`002`, `005`, and `007` are the three most implementations get wrong.

If you only run one pair, run **`008` fragments A and C**. They are identical apart from
where sampling stopped, every recorded sample in both is `true`, and an evaluator that
returns the same verdict for them has a silent second route to the defect `002` exists to
catch.

Before running anything, check one structural thing: **does your evaluator take the
specification as an input, or only the bundle?** One that reads only the bundle cannot pass
`007` at all — it has no way to know an assertion is missing.

## Reading a case

`spec.md` states the distinction under test and what each fragment must return, with the
reasoning. `bundle.json` is the fixture. `expected.json` is the answer key, and its `why`
fields are the argument, not decoration — if you disagree with a verdict, that is where the
disagreement is.

`002` was written first because it is the case a real suite got wrong.

Several bundles contain deliberate traps — things an evaluator must **not** use: executor
`verdict_hint` fields (`003`, `006`), a recorded `result: true` that is not honored (`004`),
and response fields that would let a skipped assertion be re-derived (`007`, marked
`_trap`). An evaluator that reaches the right verdict by using any of them is right by luck,
and will be wrong on the next bundle.

## Contributing a case

See [`../../CONTRIBUTING.md`](../../CONTRIBUTING.md). A case is `(spec fragment, bundle,
expected verdict)` plus one sentence naming the mistake it catches. The most valuable are
those encoding a false green from [`12-false-greens.md`](../../method/12-false-greens.md) —
that is how a catalogue entry stops being advice and becomes mechanically checkable.

**These are data, not tests of any particular runner.** They constrain behavior, which is
what lets a second implementation exist.
