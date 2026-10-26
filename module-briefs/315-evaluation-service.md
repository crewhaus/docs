# evaluation-service

Status: implemented and tested
Dependency phase: 9 - Telemetry & eval
Catalog layer: R15 - Telemetry, Tracing, Eval
Origin in ordering: 0.8.0 — the provability release (design §4.4); retires the in-loop judge's four hand-kept copies
Workspace home: packages/evaluation-service
Targets: CLI, CHN, MGD (every emitter whose IR carries `evaluation:` or `provability:`), plus the `crewhaus run` / `serve` interpreter and the eval runner
Test layers: T1, T3, T8

## Purpose

The composition root for the in-loop gates — what `memory-service` did for the memory fabric and `model-service` did for routing, applied to turn completion. One call, `wireEvaluation(fragment, deps)`, builds the `evaluation:` judge, the `provability:` gate and the proof tools from the lowered IR. Every emitter renders that call and the interpreter makes it directly, so the gates are built in exactly one place and runtime-core receives them as closures (`RunEvaluation`, `RunProvability`).

Before this package the in-loop grader was built in four places that had to agree — `renderEvaluation` in target-cli, target-channel-bot and target-managed, and `buildRunEvaluation` in `apps/cli` — and the provability gate would have been the fifth. The 0.6.0 rule now covers gates: a new in-loop gate lands here, never in an emitter's codegen.

Not to be confused with the planned `eval-service` (brief 109), which would run offline suites. This package wires the gates a live turn passes through.

## Boundaries

Owns:

- **`wireEvaluation`** — returns `{ evaluation?, provability?, tools, proofTools }`. `evaluation` is exactly the judge the retired copies built: `llm_judge` through eval-judge's panel (repeats, temperature, target, profile params; spend on the run's bus as `role: "judge"`), `contains` / `regex` pure. `provability` is the gate: its `verify` runs provability's `verifyDocument` over the final text against an evidence index built from the session event log (sub-agent children included) merged with the transcript, scoped by `scope`; `renderBundle` is the footer or the JSON, `nudge` the retry message, `persist` the bundle store, `displayPath` the footer's path rule. Nothing is opened at wiring time: every root is resolved per turn for the active tenant (`@crewhaus/tenancy`), so one wiring at a managed daemon's boot serves every tenant, fenced.
- **What the verifier needs from outside the process.** provability reaches nothing itself; this package injects the three callbacks. **Replay** re-runs pure tools through the run's catalog, bounded and abortable, and never reads as a mismatch a tool that cannot reproduce (one that read the clock, or whose answer was about the workspace as it was). **HEAD** under `sources.resolve: head` goes through tool-fetch's `guardedHead`, one per URL per session; each URL is screened by the run's egress fabric first, and each decision is recorded as an `egress_decision` with sink `provability.resolve`. **The waiver judge** (`provability.judge`, opt-in) is one sentinel-wrapped yes/no call per waivable paraphrase miss, at temperature 0, metered as `role: "judge"`, and skipped — not guessed — once `budget.judge_share` is spent.
- **The bound proof tools** — `ProvabilityPreflight` always; `ProofCheckers` and `ProofCheck` at `proven`, or when the lowered `formal.checkers` is not the default `[builtin, smt]`. All are built through tool-proof's `createProofTools` with the gate's configuration, proofs root and evidence builder, so the tools and the gate agree. `registerGateTools(catalog, gates)` and `mergeGateTools(tools, gates)` replace a same-named builtin the spec listed, because the unbound instance files its artifacts where the gate never looks.
- **The footer target** — `"stream"` on the cli (the footer prints after the answer, and a rejected attempt is marked ✗ under it before the re-run), `"reply"` on channel bots and `crewhaus serve` (appended to the text that is sent), `"none"` on the managed gateway and the eval bridges (the bundle rides the caller's own envelope through `onBundle`). `observeProofBundles` gives one run's bundle to a surface that wires once and serves many runs.
- **The `provability` skill** — `provabilitySkills(block)` renders `default-skills`' `provability` skill from the block (the floor, the checker order, and every setting that changes what this gate accepts) as a lowest-precedence builtin, so a user or project skill of the same name replaces it. `provabilitySkillText` gives a Task child the same text, so a researcher child cites the way the parent's gate reads; only the parent's final text is gated.
- **The codegen twins** — `renderEvaluationWiring(ir)` renders the one boot line (`const __gates = await wireEvaluation(<fragment JSON>, { … })`), the `evaluation:` / `provability:` fields and the tool registration; `renderGateSkills(ir)` renders the skill. Both render nothing when the IR has neither block.
- **`proofsDirName`** — where a spec's proofs live (`.crewhaus/proofs/<name>/`), mapping a name provability's store would refuse to its safe characters plus a short hash, so any name the spec schema admits wires. Everything that reads the proofs tree uses it.

Does not own: the verifiers, the bundle and its renderers (`provability`); the gate's lifecycle — ordering (provability before the judge), retry / halt / note / escalate, the exit code, the `provability_graded` line and the `(model-written, not the verdict)` mark on an answer line shaped like the footer all live in `runtime-core`; the judge's prompt and panel (`eval-judge`); the tools' behaviour (`tool-proof`); the checkers (`proof-checker`); and the research shape's report gate, whose bundle calls `verifyDocument` through `report-writer` and imports only this package's HEAD resolver and waiver judge.

## Inputs and Outputs

Inputs: an `EvaluationFragment` — `{ specName, primaryModel, evaluation?, provability? }` — which is data: an emitter serializes it into the boot line, and the interpreter builds the same object from the same IR (`evaluationFragmentFromIr`, which returns `undefined` when neither block is declared). Plus the deps that exist at the call site: `cwd`, the catalog replay finds tools in, `sessionRootDir`, `footerTarget` / `streamed`, a sandbox for the sandbox checkers, `resolveHead`, `onBundle`, `proofsRoot` (the eval runner points it into each sample's own root) and the test seams.

Outputs: the `RunEvaluation` and `RunProvability` closures for `runChatLoop`, and the proof tools for the run's catalog or tool list.

## Dependency Notes

Depends on `@crewhaus/provability`, `@crewhaus/proof-checker`, `@crewhaus/tool-proof`, `@crewhaus/eval-judge`, `@crewhaus/runtime-core` (the gate types), `@crewhaus/default-skills` and `@crewhaus/skills-registry` (the skill), `@crewhaus/model-service` (the judge's instrument id), `@crewhaus/tenancy`, `@crewhaus/tool-catalog`, `@crewhaus/tool-fetch` (the guarded HEAD), `@crewhaus/tool-result-store`, `@crewhaus/trace-event-bus`, `@crewhaus/ir`, `@crewhaus/errors` and `@crewhaus/logging`.

A bundle with neither block renders nothing and imports nothing, so it compiles byte for byte as before. A cli, channel or managed bundle with `evaluation:` changes shape — one `wireEvaluation` call where an inlined judge used to be — and the judge behaves as before; this is the release's one codegen-shape change, and the CHANGELOG lists it. As with `model-service`, a compiled bundle importing this package is a consumer of the workspace, not a cycle.

`ProvabilityPreflight`, `ProofCheckers` and `ProofCheck(builtin)` carry builtin `alwaysAllow` rules in permission-engine, so the gate's own nudge never parks a headless turn. `ProofCheck(smt)` and `ProofCheck(lean4)` start a container and keep the mode default, and `crewhaus lint` says so on a channel or managed spec (`provability-proofcheck-asks`).

## First Implementation Slice

Built in the release's second stage, in one piece: the four judge copies collapsed into `wireEvaluation`, with their behaviour pinned by tests rather than by a byte-diff (runtime-core's evaluation and judge-metering suites are unchanged), and the same root built the provability gate with its evidence and replay, the guarded HEAD resolver, the waiver judge, the bound proof tools and the per-tenant roots. The `provability` skill and `observeProofBundles`, through which the eval runner takes each sample's bundle, came in the third stage.

## Study References

`memory-service`'s `wireMemory` and `model-service`'s `wireModels` (the composition-root pattern), runtime-core's turn-completion wrapper (the consumer), `eval-judge` (the judge and its categorical form), and design §4.4, §5 and §8.

## Validation Plan

Catalog tests: T1, T3, T8. Primary risks: **a gate the compiled bundle and the interpreter build differently** — answered by one root, a fragment that is data, and codegen twins rendered from the same IR; **a gate the agent can feed** — the evidence comes from the log rather than the messages, the run's own state directories are named to the verifier as never-sources, and the agent's tools cannot write them (the runtime-state floor, on in every harness); and **a gate that leaks** — each HEAD is screened by the egress fabric and recorded, and the nudge carries no fetched text.

Definition of done: tests green; the three acceptance scenarios of design §0 — the fabricated citation, the hand-computed total, the overlap claim — un-servable through the live loop with the model misbehaving (`acceptance.test.ts`); and a spec with neither block emitting nothing.
