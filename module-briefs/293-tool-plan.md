# tool-plan

Status: implemented and tested
Dependency phase: 11 - Built-in tool implementations
Catalog layer: R4 - Built-in Tool Implementations
Origin in ordering: v0.3.0 memory release (design §2.4/§7.4, PR 7)
Workspace home: packages/tool-plan
Targets: CLI, CHN, MGD, RES, CRW
Test layers: T1

## Purpose

The continuity fabric's tool surface: `FocusRead`/`FocusWrite`, `PlanRead`/`PlanUpdate`/`PlanComplete`, `GoalWrite`/`GoalUpdate`/`GoalList`, and `MemoryClear`. `PlanComplete` is the proof ladder's `proven` transition — it resolves each cited `toolUseId` against the session event log and rejects unverifiable citations with an instructive error ("run the action first, then complete the step with its toolUseId"), so narration can never produce a ✓.

## Boundaries

Owns: the RegisteredTool definitions and their Pillar 3 flags — `MemoryClear` is destructive + `requireJustification: true`; `FocusWrite`/`PlanUpdate` are destructive + audit-and-allow (a judge call per routine plan update would drown the loop). Mutations emit the additive `plan_update` / `goal_update` / `action_proof` event kinds through an injected `appendEvent` seam and record `runContext.agentIdentity` (e.g. `subagent=researcher`) when set.

Does not own: storage (`continuity-store`), the `plan.dirty` re-render loop (`runtime-core` via the RuntimeBridge's `runState`), or goal mirroring to Thredz (`memory-service`'s thredz wiring, PR 16).

## Inputs and Outputs

Inputs: a ContinuityStore, the injected event-append seam, tool-call payloads.

Outputs: registered tools for the catalog; plan/goal/focus mutations; proof verdicts surfaced as tool results.

## Dependency Notes

Depends on `@crewhaus/tool-builder`, `@crewhaus/continuity-store`, `@crewhaus/errors`. Registered by `memory-service` (`wireContinuity`), honouring `continuity.plan: false` (keeps only Focus* + MemoryClear).

## First Implementation Slice

Shipped in factory PR 7 alongside continuity-store; `TodoWrite` is kept and re-backed by the plan store (working-tier steps of the active plan) rather than retired.

## Study References

`tool-todo` (the module-level-state failure this replaces), `permission-engine`'s intent gate (recipe 53); design doc §2.4 (the claimed→proven ladder).
