# grader-provability

Status: implemented and tested
Dependency phase: 9 - Telemetry & eval
Catalog layer: R15 - Telemetry, Tracing, Eval
Origin in ordering: 0.8.0 — the provability release (design §9.1)
Workspace home: packages/grader-provability
Targets: EVAL (grading the proof bundles of any gated shape's samples: `crewhaus eval` on a spec with `provability:`, or `crewhaus eval --provability <rung>` on one without)
Test layers: T1, T5

## Purpose

The provability eval layer: deterministic graders over the proof bundle the gate produced for each eval sample. They make no model call and never re-run the verifier — the bundle is the gate's own record of the answer, so a graded sample and the footer the answer carried can never disagree.

| Grader | Measures | Passes |
|---|---|---|
| `provability.claimCoverage` | claims that meet the floor ÷ claims that count (assumptions excluded) | ≥ 0.95, and every `expected_sources` origin grounds the answer |
| `provability.rungAchieved` | the highest floor the answer passed, as rank ÷ 4 | at or above the floor |
| `provability.citationValidity` | defined markers ÷ markers (the `cited` sub-check alone) | ≥ 0.99 |
| `provability.assumptionRate` | declared assumptions ÷ claims (lower is better) | ≤ 0.1 |
| `provability.replayIntegrity` | replayed tool results that matched ÷ replayed | = 1.0 (none replayed passes) |
| `provability.costPerProvenClaim` | gate-caused spend per met claim | reported, never gates |

`rungAchieved` is the one that shows *how far* an answer fell short rather than only that it did; `assumptionRate` makes declaring everything an assumption visible rather than a way through the gate.

## Boundaries

Owns:

- **The graders** — `PROVABILITY_GRADERS` under their canonical names, `PROVABILITY_METRIC_THRESHOLDS`, `registerProvabilityGraders(registry)` (which skips a name already registered, so a plugin registered first keeps it, as `registerContinuityGraders` does) and `createProvabilityGraders({ pricing })`.
- **A sample's floor and expected sources** (`metadata.ts`) — a dataset sample may carry `metadata.provability: <rung>` or `{ level?, expected_sources? }`, the string being shorthand for `{ level: <rung> }`. The floor is the sample's `level` when set, else the run's (the spec's floor, or `--provability`). `expectedSourceKey` normalizes an origin the way the bundle did, and a source the run wrote itself grounds the answer but does not count toward `expected_sources`; `claimCoverage` names it in its rationale, beside a tally of the unmet reasons.
- **Unmeasured outcomes** — four failures replace a measurement, each with its own pinned rationale prefix. A run with no bundle (it was not gated, or the sample ended before the gate graded an answer) fails every grader with "no proof bundle (is provability: declared?)". A bundle whose verifier threw, so the answer stood unverified, fails every grader with "the gate could not verify this answer". A malformed `metadata.provability` fails every grader with "invalid metadata.provability…", naming the field. A sample whose floor is above the floor the bundle was made at fails only the floor-dependent graders — `claimCoverage`, `rungAchieved`, `replayIntegrity` and `costPerProvenClaim` — with "not gradable at this floor", because rungs above a gate's floor are recorded without the checks only those floors run; `citationValidity` and `assumptionRate` do not depend on the floor and still measure. The eval runner gates each sample at its own floor, so the last case arises only for a bundle made elsewhere. Proof was expected, so a missing verdict is never a pass; `isUnmeasured(result)` matches all four prefixes, telling these failures apart from measured ones.
- **Roles** — `provabilityGraderRole(name)` gives every path that adds a grader (the `--provability` flag, a graders file's `type: registry` entry) one answer. The graders with a bar are **requirements**: a sample must pass each on top of its graders file's `combine:` policy, never as a vote inside it, so a vacuous pass or a lower-is-better rate cannot carry a `combine: any` or skew a `weighted` score. `costPerProvenClaim` is a **report**: more gate spend can never turn a fail into a pass.
- **Cost** (`cost.ts`) — the spend the gate caused, read from the run's own trace events and never estimated: model calls in a retry window after a failing pass, and the waiver judge's reported spend. Checker time is reported beside it, not added to it. A metered run's `cost_accrual` events are the price; an unmetered one is priced through the caller's `pricing` seam, and tokens nothing can price are reported as unpriced, never as free.
- **Roll-ups** (`summary.ts`) — `summarizeProvabilityMetrics`, `summarizeCostPerProvenClaim` and `renderProvabilitySummaryLines`: per-metric means, pass fractions and percentiles in the shape the continuity and 12-metric summaries take, so a report surface can treat the three rubrics alike. An unmeasured result always counts as a failure and is kept out of a lower-is-better mean, where its placeholder 0 would hide a breach.

Does not own: capturing the bundle — `eval-runner` wires the gate per sample through `evaluation-service`'s `wireEvaluation` (the production gate, with its retries, nudges, proof tools and skill), keeps the sample's proofs under the sample's own root, persists the standing bundle as `<sampleDir>/proof.json` and hands it over as `RunResult.artifacts.proof`; re-flooring a sample's gate at its metadata level, the `--provability` flag and the `combine:` arithmetic (`eval-runner`); the bundle and its levels (`provability`); the `provability.*` optimizer paths (`spec-patch`); and the advice rule that flags a harness whose answers keep failing the gate (`harness-advice`).

## Inputs and Outputs

Inputs: a `Sample` (its `metadata.provability`) and a `RunResult` whose `artifacts.proof` holds the sample's standing bundle — the gate's last pass over the final answer — plus the run's trace events for the cost grader.

Outputs: per-sample grader results with rationales, and the cross-sample summaries.

## Dependency Notes

Depends on `@crewhaus/grader-registry`, `@crewhaus/eval-grader`, `@crewhaus/eval-dataset` (the sample type), `@crewhaus/provability` (the bundle type and its levels) and `@crewhaus/trace-event-bus` (the events the cost grader reads). Deterministic by design, like `grader-continuity`: these graders read end states offline and touch none of the harness's live stores.

`eval-runner` registers them in its default registry, so a graders file can name any of them with `type: registry`, and `crewhaus eval --provability <rung>` adds all of them to a run without editing the spec — which is how a team measures the cost of a rung before adopting it. A `weight` on one of them is not used, and under `combine: any` or `weighted` the run says so at start.

## First Implementation Slice

Built in the release's third stage, with the rest of the eval layer: the graders, `RunArtifacts.proof`, `eval --provability` and dataset `metadata.provability`. That stage's review added two runner-side changes: each sample's proofs root is isolated from the harness's live proofs tree, so an eval run neither lands in `crewhaus provability audit` nor proves a claim with an artifact a production run wrote; and a sample with its own `metadata.provability.level` has its gate re-floored at that level, so its bundle is made at the floor it is graded at.

## Study References

`grader-continuity` and `grader-12-metric-rubric` (the install-and-summarize conventions), `continuity.costPerProvenOutcome` (the cost precedent), and design §9.1.

## Validation Plan

Catalog tests: T1, T5. Primary risks: **a grader that passes what the gate refused** — they read the bundle and never re-verify, so they cannot disagree with it; **a missing verdict read as a pass** — pinned "unmeasured" outcomes; and **a provability grader carrying a sample through someone else's policy** — the requirement / report roles.

Definition of done: `bun test packages/grader-provability/src` green, and each grader's pass, fail and unmeasured outcomes pinned against fixture bundles.
