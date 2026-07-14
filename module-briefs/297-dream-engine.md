# dream-engine

Status: implemented and tested
Dependency phase: 8 - Context & memory
Catalog layer: R6 - Context & Memory (triggers live in R14 - Scheduling & Background)
Origin in ordering: v0.3.0 memory release (design §6, PR 14)
Workspace home: packages/dream-engine
Targets: CLI (boot catch-up + `crewhaus dream`), CHN, MGD (janitor step)
Test layers: T1, T3

## Purpose

Scheduled memory consolidation (`memory.dream`) — consolidation on a schedule, not every turn. A dream run is two phases. Phase 1, deterministic (always; idempotent; zero model spend): fact TTL sweep + near-duplicate supersede + `compact()` growth bounding, staleness flags (facts >90d, wiki unverified >30d), sessions-index fold-in, proof-excerpt re-validation + retention-pin refresh for records citing sessions nearing TTL, focus/handoff next-actions refresh, trash purge. Phase 2, model synthesis (`mode: full` AND `budget_usd > 0`): ONE bounded fresh session seeded with the builtin `dream` skill + phase-1 findings, acting ONLY through the normal registered tools (full justification/audit path), budget-capped, and refusing to run an unpriced model.

## Boundaries

Owns: the two-phase run, `.crewhaus/dream/<spec>/state.json` (`lastRunAt`/`lastOutcome`/`phase1Counts`/`lastEvidence`), the additive `dream_run` event kind, and window idempotency (`dream:<spec>:<floor(now/every)>` through durable-execution's `withIdempotency`) so a janitor tick, a GH-Actions cron, and a CLI invocation can never double-fire — including under `fleet run` parallelism.

Does not own: the triggers themselves — runtime-core's janitor (now a pluggable step registry) registers the daemon step, the cli shape runs a boot-time deterministic catch-up, and `crewhaus dream run|status|init` ships the cron-safe verbs (apps/cli). Does not own the stores it consolidates or the `memory.dream` spec block.

## Inputs and Outputs

Inputs: the lowered dream config (`everyMs`, `mode`, `budgetUsd`, `instructions`), the wired stores (facts/wiki/continuity), a model session factory for phase 2.

Outputs: consolidated stores, the state file, `dream_run` events, a per-phase outcome report (the `crewhaus dream` checklist rendering).

## Dependency Notes

Depends on `@crewhaus/memory-store`, `@crewhaus/wiki-store`, `@crewhaus/continuity-store`, `@crewhaus/session-store` (`summarizeSessionIntoIndex`), `@crewhaus/durable-execution`, `@crewhaus/cost-tracker` (unpriced-model refusal). `CREWHAUS_DREAM=0` / `CREWHAUS_DREAM_INTERVAL_MS` env knobs follow the daemon convention.

## First Implementation Slice

Shipped in factory PR 14 (engine + janitor step registry + CLI verbs + boot catch-up); PR 17 composes learning's `study.on_dream` gap/curriculum seeding onto the existing `DreamModelPhase` seam.

## Study References

Consolidation-on-a-schedule from the 2026 memory evidence base (the design's §13 mapping table), `heartbeat-engine` (liveness vs consolidation — deliberately distinct), `security-digest` (the cron-safe CLI verb conventions); design doc §6.
