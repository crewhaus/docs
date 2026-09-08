# model-service

Status: implemented and tested
Dependency phase: 3 - Model & Tool primitives
Catalog layer: R2 - Model Layer
Origin in ordering: v0.6.0 model-plan release (design §2 stance 4, §7; factory PRs #430, #432, #437, #441, #444)
Workspace home: packages/model-service
Targets: All (pool-bearing shapes wire hybrid closures; the rest get profile-resolved model/params for free)
Test layers: T1, T3

## Purpose

The composition root for model routing, mirroring what `memory-service` did for the memory fabric: one call, `wireModels(fragment, deps)`, turns the lowered routing IR into a spread-ready `RunChatLoopOptions` fragment. Emitters stay dumb — typed IR in, one stable call out — and adding a routing feature means extending the fragment and this root, never re-templating fourteen emitters.

It resolves adapters and per-profile failover chains and breakers, opens the scoreboard and the priors, constructs the router with its rules, classifier and eligibility filter, wires the judge grader to the run bus so judge spend is priced, and builds the hybrid closures — `Consult` / `Escalate`, the classifier label call, and the guide / shadow / committee lanes.

## Boundaries

Owns:

- **`wireModels`** — the one call every emitter and the `crewhaus run` interpreter makes. It is per RUN, not per process: the escalation latch it returns is bounded per run, so a host serving many runs from one process calls it once per run and never caches the fragment.
- **`wireHybrid` and its codegen twin `renderHybridWiringFields`** — the runtime closures cannot ride the pool blob an emitter serialises, so a bundle whose lowered pool declares `strategy.model_directed`, `policy: classifier` or `strategy.{guide,shadow,committee}` renders the call beside the literal routing fields and imports this package at boot. The interpreter and the compiled bundle therefore construct exactly the same closures. A pool that declares none of the three renders nothing and imports nothing, so every pre-0.6.0 bundle is byte-identical.
- **`HYBRID_FAMILIES_BY_SHAPE`** — the per-shape carry/emit/ignore matrix, in code. Shape → the closure families that shape's bundle constructs, read by the emitters, by the compiler's warning, and by `crewhaus models explain`. One table, three consumers, so the published matrix and the emitters cannot drift.
- **The side calls** — guide, shadow and committee, all routed through one nested single-turn runner: a tool-less `runChatLoop({ singleTurn: true })` with its own child context and bus whose model events are re-published on the parent's under the side call's role and stage. The parent prices them, meters them under `budget.judge_share`, and persists them; the child writes no session file of its own.

Does not own: the plan arithmetic (`model-plan`), the tools themselves (`tool-consult`), the scoreboard's line format (`routing-store`), or any lifecycle decision — when a stage runs, what its text does, and how the outcome folds into the arms all stay in `runtime-core`, which remains store-free.

## Inputs and Outputs

Inputs: the lowered routing fragment (`modelFallbacks`, `modelTiers`, `modelPool`, `circuitBreaker`), plus deps — the adapter factory, catalog, run bus, scoreboard root, session name, and any `--model` override.

Outputs: a `RunChatLoopOptions` fragment carrying the router, the per-candidate adapters, the hybrid tools and the side-call closures — spread into the loop options by the caller.

## Dependency Notes

Depends on `@crewhaus/ir` (types), `@crewhaus/adapter-anthropic`, `@crewhaus/cost-tracker`, `@crewhaus/eval-judge`, `@crewhaus/run-context`, `@crewhaus/runtime-core`, `@crewhaus/tool-catalog`, `@crewhaus/tool-consult` and `@crewhaus/trace-event-bus`.

A compiled bundle importing this package is not a dependency cycle: the bundle is a *consumer* of the workspace, not a member of it, and the emitted `dist/package.json` lists the import because the manifest is collected from the bundle's own import strings.

An explicit `--model` override suppresses the pool, the tiers and the fallbacks — one model was asked for, so one model serves.

## First Implementation Slice

Landed as a refactor first (PR #430 wrapped the existing per-emitter routing fragments byte-identically), then grew: the model-directed pair (#432), the side calls (#437), bundle reach for the closure families (#441), and the remaining four pool-bearing shapes plus the in-code matrix (#444).

## Study References

`memory-service`'s `wireMemory` (the composition-root pattern this copies), `model-router`'s policy router and failover chain (what it constructs), and `runtime-core`'s per-candidate plan table (its consumer).

## Validation Plan

Catalog tests: T1, T3. Primary risks: a **byte drift** on specs that declare no pool (guarded by the wrap-first discipline and per-shape smoke fixtures), and a **reach gap** — a bundle that has the pool but not the behaviour, which is exactly what the in-code shape matrix and its emitter tests exist to prevent.

Definition of done: tests are green, an absent hybrid strategy emits nothing, the interpreter and the compiled bundle build identical closures, and every shape's emitter agrees with `HYBRID_FAMILIES_BY_SHAPE`.
