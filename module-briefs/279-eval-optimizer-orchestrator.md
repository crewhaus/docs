# Module brief 279 — `eval-optimizer-orchestrator`

**Catalog layer:** F-eval.
**Pillar:** Pillar 2 — closes the active optimisation loop.
**Source:** [`packages/eval-optimizer-orchestrator/src/index.ts`](https://github.com/crewhaus/factory/blob/main/packages/eval-optimizer-orchestrator/src/index.ts).

## Responsibility

The glue that ties `prompt-optimizer` (search), `spec-patch` (structured mutation), and a caller-supplied fitness function (typically backed by `eval-runner`) into a single user-facing workflow. Drives the `crewhaus optimize` CLI subcommand.

Without this package, `prompt-optimizer` is an orphaned search function with no way to turn its winning candidate back into a YAML change the user can commit.

## Public surface

```ts
export type OptimizeSpecOptions = {
  readonly specPath: string;
  readonly fitness: FitnessFn;
  readonly trainSet: ReadonlyArray<Sample>;
  readonly devSet: ReadonlyArray<Sample>;
  readonly mutator?: MutationProvider;
  readonly iterations?: number;
  readonly improvementThreshold?: number;
  readonly outDir?: string;
  readonly writeBack?: boolean;
  readonly runId?: string;
  readonly seed?: number;
  readonly budgetUsd?: number;
  /** D43 — numeric dials the search may step alongside the rewrite. */
  readonly knobs?: ReadonlyArray<KnobDial>;
  /** D36 — the ONE stage path the search rewrites (multi-stage specs). */
  readonly promptPath?: ReadonlyArray<string>;
};

export type OptimizeSpecResult = {
  readonly runId: string;
  readonly scoreBefore: number;
  readonly scoreAfter: number;
  readonly improvement: number;
  readonly applied: boolean;
  readonly patch: SpecPatch;
  /** The prompt patch, then one patch per moved knob dial. */
  readonly patches: ReadonlyArray<SpecPatch>;
  readonly patchedYaml: string;
  readonly writtenTo?: string;
  readonly outDir: string;
  readonly trajectory: ReadonlyArray<Candidate>;
};

export async function optimizeSpec(opts: OptimizeSpecOptions): Promise<OptimizeSpecResult>;

// Multi-stage enumeration (D36) — every path returned is already whitelisted.
export {
  MULTI_PROMPT_TARGETS,
  findStage,
  formatStageNames,
  listOptimizableStages,
  stagePathIsWhitelisted,
} from "./stages";

// Budget metering + the failure arbiter
export { BudgetMeter, actualCallMicros, estimateCallMicros } from "./budget-gate";
export { aggregate, arbitrate } from "./failure-arbiter";

// Re-exports for caller convenience
export { applySpecPatch, validatePatch } from "@crewhaus/spec-patch";
export type { SpecPatch } from "@crewhaus/spec-patch";
```

## Behaviour

1. Reads the source `crewhaus.yaml`.
2. Resolves the prompt to rewrite. Without `promptPath` this is `extractCurrentPrompt(spec)` targeting `["agent","instructions"]` — the historical behaviour, byte-identical down to `report.json`. With `promptPath` it reads one named stage: a workflow step (`["steps","0","instructions"]`), a graph node (`["nodes","plan","instructions"]`) or a crew role (`["roles","writer","instructions"]`). A path that does not name a *string* instructions block is refused before any spend, pointing at `listOptimizableStages(spec)`.
3. Validates any declared `knobs` against this target's `OPTIMIZABLE_PATHS` before anything is spent, and refuses a knob-blind `fitness` (a single-parameter fitness cannot apply the dials it is being asked to measure).
4. Drives `prompt-optimizer.optimize()` with the caller-supplied fitness function, metering model spend through `BudgetMeter` when `budgetUsd` is set.
5. Converts the winning prompt into a `SpecPatch` against the resolved path, plus one patch per moved knob dial.
6. Validates every patch via `validatePatch` (cross-target + `OPTIMIZABLE_PATHS` check).
7. Applies them via `applySpecPatch` (CST round-trip — comments preserved).
8. Persists `patch.json`, `patches.json`, `report.json`, `trajectory.json` and `best.json` under `outDir`.
9. Optionally writes the patched YAML back to disk with a leading header comment when `writeBack: true` and `improvement >= improvementThreshold`.

## Boundaries and stated limits

- **Multi-stage specs are supported.** The old "only `agent.instructions` is mutated" limit is gone: `workflow` / `graph` / `crew` / `pipeline` specs optimise per stage. A multi-stage spec with no stage named raises `OptimizeSpecError` listing the spec's **actual** stages and pointing at `crewhaus optimize --stage <name>` (or `promptPath` on this seam) — it no longer points at a `--path` flag, which `OPTIMIZE_SCHEMA` never defined. `kind: judge` steps and nodes run no agent turn and are never mutated.
- **Knob search is library-only.** `knobs` reaches the bounded coordinate-ascent `knob-step` mutation in `prompt-optimizer`, gated by the same fitness accept loop. There is **no `--knobs` CLI flag**, so a `crewhaus optimize` or `flywheel run` invocation proposes no knob changes today.
- The fitness function stays the caller's responsibility. The CLI wraps `eval-runner.runEval()` — and, for a bridged multi-stage candidate, the same eval-entry emission `compile --with-eval-harness` performs, driven through the shared bridge invoker — so the package itself stays decoupled and test harnesses can supply synthetic fitness.
- The CLI's sample projection into the search is **narrower** than the eval gate's: `expected_tools` and `metadata` are stripped during the search, and `history`-carrying samples are refused up front on bridged non-chat-capable shapes. Regression pinning still pins the original un-stripped records, so `crewhaus eval` grades them in full at the gate.

## Depends on

- `@crewhaus/errors`
- `@crewhaus/eval-dataset` (Sample type)
- `@crewhaus/prompt-optimizer` (114) — search loop
- `@crewhaus/spec` — parser
- `@crewhaus/spec-patch` (278) — mutation

## Unblocks

- `crewhaus optimize` CLI subcommand, including `--stage` and `--mutator meta-harness`
- `crewhaus flywheel run` (which composes it behind the acceptance gate)
- Future Studio panel for "Optimise this spec"
- Plugin-extensible optimisation paths (different `MutationProvider`s)

## Tests

- `index.test.ts` — synthetic fitness, score delta, patch shape, persistence, write-back CST round-trip, sub-threshold `applied: false`
- `stages.test.ts` — per-stage enumeration, whitelist membership of every returned path, the named-stages error message
- `budget-gate.test.ts` — the spend meter and its stop reasons
- `failure-arbiter.test.ts` — failure aggregation/arbitration

## Risk markers

🟡 (medium risk) — depends on prompt-optimizer's search loop semantics. If the search loop changes shape (e.g. introduces parallel candidates), the orchestrator's `result.improvement` arithmetic needs to be re-derived. The trajectory test asserts the current contract. Multi-stage runs add a second contract worth pinning: stages compose **sequentially and are gated independently**, so a losing stage must leave the working spec untouched for the next stage to start from.
