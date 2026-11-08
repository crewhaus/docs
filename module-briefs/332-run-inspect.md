# run-inspect

Status: planned
Dependency phase: 9 - Telemetry & eval
Catalog layer: R15 - Telemetry, Tracing, Eval
Origin in ordering: 0.9.0, the Home release ([#518](https://github.com/crewhaus/factory/issues/518))
Workspace home: packages/run-inspect
Targets: All (it operates compiled harnesses; it is not embedded in one)
Test layers: T1, T3, T4

## Purpose

Answers "why did my harness do that?" from a run's records: what started the run, what each model call was given, which rules were in force, which checks ran, who decided, and what happened as a result. It joins records that each store keeps apart, and shows what is missing instead of dropping it.

It is pure folds with no state of its own. Two heads need the same folds: `crewhaus runs inspect` in the terminal and the run inspector in Home, the 0.9.0 name for Hangar's web console. So they live below `hangar-server` and outside `control-service`'s tree, because the CLI may import from hangar-server only its start surface, and because `harness-supervisor` owns processes, not explanations. Inspection stays deterministic: a model-written explanation belongs to Home's assistant, in its own panel, never here.

## Boundaries

Owns:

- **The read model.** Pure, deterministic folds over a run's session log (an eval sample's transcript is one), the supervisor's execution events for that run, the run records and decision log, proof bundles, feedback, and audit records by run id:
  - `InspectSummary`: a status word, what started the run, what was asked, a templated "what happened", the outcome, cost, duration, revisions, parent and child runs, missing evidence;
  - `TimelinePage`: entries nested under their spans, key events or everything recorded, paged newest first, with a line wherever a cap cut the list;
  - `SpanTree`: the durable tree of turns, model calls and tools;
  - `WhyRecord`: for a tool call, answer, model call or turn, what started it, what it used, the rules in force, the reason it gave, every check, and who decided (a member, an agreement's fallback or an owner's delegation rule, with its evidence);
  - `ContextUsed`: per model call, what was included beside what was available, a redacted memory staying visible as "Hidden for safety" with its rules, never its text;
  - `RequestView`: the receipt, the masked request body and the run it created;
  - `RawView`: masked raw lines, every thinking block annotated.
- **Field provenance.** Every field says how it is known: Recorded, Pieced together (by a stated rule), Not recorded (with the reason) or Generated. Joins use explicit keys, the run and the span or the run and the call's position. A fallback join is labelled Pieced together.
- **Missing evidence.** A closed list of reasons (`before-0.9`, `records-off`, `advisor-off`, `cost-off`, `not-supervised`, `pruned`, `evicted`, `truncated`, `torn`, `unfinished`, `diverged`, `no-access`, `unsupported`, `not-recorded`), each with its range, count and remedy. A session written before 0.9.0 reads as one legacy run, with a note that its runs cannot be told apart.
- **Cost, honestly.** It sums the per-call cost lines and the roll-ups sub-agents publish, never the role-less run total, so nothing counts twice. An unpriced call makes the total "Cost unknown", never `$0`.
- **Evidence, not reasoning.** Model thinking never appears in a decision view or an export. The raw view labels it, and the operation that serves it needs the owner.
- **The importance taxonomy** that picks key events, and **the inspect projection fold**, per-run facts for fleet lookups that the projector of brief 325 builds off the request path under the Hangar root's `cache/index/`. No GET builds it, and deleting it rebuilds the same answers.

Does not own: writing the records (`runtime-core`, `event-log`, `trace-event-bus`, extended in place by [#518](https://github.com/crewhaus/factory/issues/518)); supervised processes and the run-detail reader (`harness-supervisor`, brief 305); proof bundles (`provability`, brief 316); knowledge and people lookups, injected by the caller (`knowledge-index`, brief 335; `people`, brief 328); the `crewhaus.hangar.v1` operations, masking and authorization (`hangar-server`, brief 301); the screens, Play back and Run again (`hangar-ui`, `control-service`).

## Inputs and Outputs

Inputs: a canonical run id or session id, resolved by the caller to files inside their roots; bounded readers for those files; an optional knowledge `lookup` and an optional people-domain check, both injected; and the requested view, cursor and limit.

Outputs: the seven view types, as plain data with their missing-evidence lists, for a head to mask and render. The CLI is the first consumer: `crewhaus runs inspect <run or session>` prints the summary and key events, with `--timeline [--full]`, `--why <action>`, `--context [<call>]`, `--request`, `--raw`, `--export <file>`, `--work <item>` and `--json`.

Run stamps, context manifests, spans and verdict lines are on by default once a bundle is recompiled, and `CREWHAUS_RUN_STAMPS=0` turns them off. With them off, or on a bundle built before 0.9.0, this package reports the matching missing-evidence reason. Masking is the caller's: every v1 response passes the dispatch-site masker and the terminal twin masks the way `crewhaus daemon logs` does, and output field names pass the masker's key rule, so no value is redacted by accident.

## Dependency Notes

Depends on `@crewhaus/control-contract`, `@crewhaus/control-records`, `@crewhaus/session-store`, `@crewhaus/event-log`, `@crewhaus/provability`, `@crewhaus/feedback-distill` and `@crewhaus/harness-supervisor`'s `./run-detail`, which is on main. It is light, uses only `node:` APIs, never spawns, never calls a model and writes nothing, so a read leaves a git-clean harness clean. It adds no spec key, IR field, TrustOrigin or emitted byte, and no third-party dependency.

## First Implementation Slice

`feat(run-inspect): explain a run from its records, starting in the terminal` adds the package and `crewhaus runs inspect`, after the runtime-core and trace-event-bus changes that write the records. Later PRs extend it: the inspector over `crewhaus.hangar.v1`, then Home's view. This brief lands in crewhaus/docs before the package does.

## Study References

`packages/harness-supervisor/src/run-detail.ts` (the run view it reads); `packages/hangar-server/src/sessions.ts`, `runs.ts`, `models.ts` and `proofs.ts` (the per-store readers whose join belongs in one package); `apps/cli/src/hangar-server-imports.test.ts` (the guard that puts it below the server); `apps/cli/src/incident.ts` (matching audit records by time window, the fallback it labels); and [#518](https://github.com/crewhaus/factory/issues/518).

## Validation Plan

Catalog tests: T1, T3, T4. Primary risks: **a confident answer built on a guess**: every field carries its provenance, and a fallback join is labelled; **evidence silently dropped**: fixtures produce every missing-evidence reason, and a cut list says so; **a verdict pinned to the wrong call**: each check in a concurrent tool group folds under its own tool span; **thinking in a decision view**: a test asserts no view but `RawView` carries it.

Definition of done: `bun test packages/run-inspect/src` green; golden folds cover every shape; `crewhaus runs inspect` leaves a git-clean harness clean; an unpriced call reads "Cost unknown" in every view; and a test holds the package free of `node:child_process` and of any model adapter.
