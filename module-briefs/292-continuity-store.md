# continuity-store

Status: implemented and tested
Dependency phase: 8 - Context & memory
Catalog layer: R6 - Context & Memory
Origin in ordering: v0.3.0 memory release (design §2, PR 7)
Workspace home: packages/continuity-store
Targets: CLI, CHN, MGD, RES, CRW (the agent-loop shapes; default-on via `continuity:`)
Test layers: T1, T3

## Purpose

The persistence substrate for v0.3.0 Goal 1: focus, plans, goals, and the teardown handoff as human-readable, user-clearable artifacts under `.crewhaus/state/<spec>/` — marker-gated `focus.md` (capped body + the verbatim `REQ-nnn` requirements ledger + active-plan pointer), `plans/plan-NNNN-<slug>.md` (YAML frontmatter + numbered steps), `goals.yaml`, and a deterministic `handoff.md` render (no model calls).

The `open → in_progress → claimed → proven` proof ladder is machine-checked here: a `proven` transition requires `toolUseId` evidence resolved against session event logs (sub-agent child sessions included via `sub_agent_start` brackets), pins the cited sessions in `.crewhaus/retention.json`, and freezes `{toolName, inputHash, resultDigest}` excerpts so evidence outlives the transcript TTL.

## Boundaries

Owns: the store layout and its atomic tmp+rename writes under the §7.6 advisory `.lock` (wait 2 s → steal >30 s stale with a warning → fail naming the holder pid; the lock helper itself now lives in `infra-utils`); the proof-ladder verification against event logs; trash-with-undo clearing through `.crewhaus/trash/<ts>/` (`moveToTrash` is exported for other stores — never a hard delete); session-scoped stores and fail-closed tenant path fencing following the session-store rules.

Does not own: the tool surface (`tool-plan`), the runtime injection seams (`runtime-core`'s `RunChatLoopOptions.continuity`), the spec/IR `continuity:` block (`spec` / `ir` / `compiler`), or the composition wiring (`memory-service`).

## Inputs and Outputs

Inputs: store roots (cwd, spec name, optional tenant/session scope), plan/goal/focus mutations, proof references (`toolUseId[]` + a session-log reader).

Outputs: the on-disk artifacts above; rendered pickup/handoff blocks; proof verdicts (`verified` / instructive rejection); `plan_update` / `goal_update` / `action_proof` payloads through an injected `appendEvent` seam.

## Dependency Notes

Depends on `@crewhaus/errors`, `@crewhaus/infra-utils` (file lock), `@crewhaus/event-log` (types + reading for proof checks). No runtime-core dependency — closures are constructed in `memory-service` and injected (the #53 pattern).

## First Implementation Slice

Shipped in factory PR 7 of the v0.3.0 train (packages only), wired by PR 10 (memory-service), threaded spec→IR→emit by PR 11, and consumed read-only by sub-agents in PR 13.

## Study References

`session-store` (path fencing + reducer precedent), the Theanine supersede-never-delete discipline, HiAgent (subgoal-organized working memory); design doc `factory/design/0.3.0-memory-release.md` §2.
