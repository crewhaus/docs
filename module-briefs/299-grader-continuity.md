# grader-continuity

Status: implemented and tested
Dependency phase: 9 - Telemetry & eval
Catalog layer: R15 - Telemetry, Tracing, Eval
Origin in ordering: v0.3.0 memory release (design §7.3, PR 19)
Workspace home: packages/grader-continuity
Targets: EVAL (grading any memory-carrying shape's samples)
Test layers: T1, T5

## Purpose

Memory quality made measurable — evaluate end states, not answers. Five DETERMINISTIC (no-LLM) graders computed from a sample's session artifacts (the eval-runner's isolated per-sample session JSONLs + `.crewhaus/` state root), installable with one call, `registerContinuityGraders(registry)`, exactly like the 12-metric rubric: `continuity.reAskRate` (gate 0 — the release's motivating failure), `continuity.reqRetention`, `continuity.proofHonesty` (a done-claim without a verified `action_proof` event scores 0), `continuity.pickupSuccess` (does session 2 act on the handoff), and `continuity.costPerProvenOutcome` (USD per verified proven step, Infinity-safe).

## Boundaries

Owns: the five graders, the cross-sample roll-ups (`summarizeContinuityMetrics`, p50/p95/p99 + pass fractions + threshold breaches) matching the rubric's summarize shape, `renderContinuitySummaryLines`, and the worked two-session discrimination fixture (`CONTINUITY_FIXTURE_SAMPLES` + a scripted mock-adapter invoker that plays conversations through the REAL event-log/continuity-store/tool-plan code paths).

Does not own: sample artifacts (the runner stamps `RunResult.artifacts` — sampleDir/sessionId/transcriptPath/stateRootDir/specName), registry resolution (eval configs opt in BY NAME via the `type: registry` grader entry against `RunEvalOptions.graderRegistry`), or the cross-backend conformance suite (`memory-service`'s `runWikiBackendConformance`, same PR).

## Inputs and Outputs

Inputs: a finished sample's transcript + state-root artifacts (never live stores).

Outputs: per-sample [0,1] grades with rationales; roll-up summaries and report lines.

## Dependency Notes

Depends on `@crewhaus/grader-registry`, `@crewhaus/eval-grader`, `@crewhaus/event-log` (types), `@crewhaus/cost-tracker` (types for `cost_accrual`). Deterministic by design — these graders gate the default-on continuity flip in CI, so they must not cost model calls.

## First Implementation Slice

Shipped in factory PR 19: the discrimination matrix is pinned end-to-end through `runEval` — one clean run passes every gate, a re-asker fails re-ask/retention/pickup, a claims-without-proof run fails honesty.

## Study References

`grader-12-metric-rubric` (the install + summarize conventions), the design's "evaluate end states" evidence row (§13), the motivating failure (§0); design doc §7.3.
