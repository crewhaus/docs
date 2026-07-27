# target-eval-bundle

Status: implemented and tested
Dependency phase: 2 - IR & Compiler
Catalog layer: F2 - Compiler & Codegen
Origin in ordering: inferred from catalog layer F2; the runtime-invoking bridge landed in the 0.4.x evals campaign
Workspace home: packages/target-eval-bundle
Targets: **EVAL**
Test layers: T1, T5

## Purpose

Codegen for the eval harness: dataset loader, runner, graders, report writer. It also owns the **bridge** `crewhaus compile --with-eval-harness` emits — a `target: eval` bundle projected from another shape's lowered IR into `<out-dir>/eval/`, so a non-`cli` shape can consume its own distilled feedback through eval / optimize / flywheel.

## Boundaries

Owns the emitted bundle and the bridge runtime helpers. The bridge is **runtime-invoking**, not an impersonation: `createBridgeInvoker` drives the source shape's actual compiled runtime — a workflow's step sequence end-to-end (the final step's output is graded, step trace events land in `RunResult.events`), a graph to `run_done` on the per-sample RunContext (a HITL pause fails the sample loudly), one crew turn through the compiled orchestrator, a pipeline's indexed agent plus Retrieve tool, a channel bot's real `runTurn` over a loopback delivery, and the managed gateway's `runOneTurn` dispatcher under an isolated per-sample tenant. The remaining shapes run their agent and its real wired tools through the single-turn loop, and the compile prints the chosen strategy plus its named fidelity gaps.

Also owns `guardHistorySamples` — sample `history` seeds only the chat-capable shapes (channel / managed / voice / pipeline), and a history-carrying sample against any other bridged shape fails **loudly at dataset load** rather than silently seeding a conversation into a runtime that consumes one trigger input — and `emitSourceBundleWithEvalEntry`, which adds the exported `runForEval` entry (workflow / graph / pipeline; their CLI `main` becomes `import.meta.main`-guarded) or an `eval-entry.ts` (crew / channel) the bridge imports.

Does not own: the grading itself (`eval-runner` / `eval-grader` / `eval-judge`); the shapes' own emitters, which it depends on rather than reimplements; `crewhaus eval`'s CLI surface, which remains `target: cli` only.

The emission is gated: a plain `compile` stays byte-for-byte identical, including the 0.3.0 `continuity: false` byte-restore contract, and the channel bundle's two extra seams — `AgentConfig.fabricRoot` (per-sample memory/continuity isolation) and the `_adapter` scripted-provider hook — appear only under the flag. `--emit-as cf-worker` is rejected with the bridge, and so is a `cli` spec (use `crewhaus eval` directly).

## Inputs and Outputs

Inputs: canonical IR plus target/profile options; for the bridge, the source shape's IR and its selected invoker strategy.

Outputs: generated bundle files, manifests, and packaging metadata — including a pinned `package.json` so a bridged run's `run.json` carries the same reproducibility manifest a `crewhaus eval` run does.

## Dependency Notes

Depends on `compiler-core`, `eval-runner`, `dataset-registry` and `grader-registry`, and — since the bridge drives real runtimes — on the four multi-stage target emitters plus `target-channel-bot` and `tenancy`.

## First Implementation Slice

Shipped long before 0.4.x as the `target: eval` emitter. The evals campaign lifted the `per-step eval bridges are not yet supported` rejection for `workflow`, `graph`, `crew` and `pipeline`, and replaced the single-turn-chat projection with the runtime-invoking bridge described above.

## Study References

`lm-evaluation-harness/lm_eval/`, `helm/benchmark/`, `ragas/.../evaluate.py`

Research focus: Task spec format; reproducibility; fidelity of a bridged runtime versus the deployed one.

## Validation Plan

Catalog tests: T1, T5. Primary risks: quality regressions that only show up through eval or trace grading, and **bridge fidelity** — a bridge that grades something subtly unlike production is worse than no bridge, so each shape's strategy names its gaps at compile time and the byte-identical plain-compile contract is pinned in tests against pre-change emitters.

Definition of done: tests are green, public types are exported from the intended workspace, failure modes use typed `CrewhausError`-style errors where applicable, and the catalog status can be updated without hand-waving.
