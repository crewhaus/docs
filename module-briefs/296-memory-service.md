# memory-service

Status: implemented and tested (supersedes the phase-8 planning brief 098)
Dependency phase: 8 - Context & memory
Catalog layer: R6 - Context & Memory
Origin in ordering: v0.3.0 memory release (design §1/§10, PR 10); closes MODULE-CATALOG-STATUS critical-path #2
Workspace home: packages/memory-service
Targets: CLI, CHN, MGD, RES, CRW (every memory-carrying emitter + the `crewhaus run` interpreter)
Test layers: T1, T3, T5

## Purpose

The memory fabric's composition root — the design's ruling principle 1 ("one composition root instead of N copies of codegen"). `wireMemory(fragment, {catalog, cwd, tenant?, sessionScope?, appendEvent?, embedder?, examRunner?})` takes one serializable fragment (the shape the §9 IR lowers into) and does everything per-emitter memory codegen used to template: constructs the stores (file backends, or Thredz over the already-connected McpHost client), registers the tools (Remember/Recall/MemoryForget; Focus/Plan/Goal/MemoryClear; the ten thredz-vocabulary `wiki_*` tools), merges the builtin skills + slash commands at lowest precedence, and returns spread-ready `RunChatLoopOptions` seams — recall/onCapture, loadPlan/onPlanDirty/onHandoff, with scope and tenant fencing passed through to every store.

Every emitter's memory codegen collapses to this one stable call; emitters stay dumb (Pillar 1) and runtime-core stays store-free (injected closures, the #53 pattern).

## Boundaries

Owns: store construction and backend selection (`file` | `thredz`); tool registration incl. `log_knowledge_gap` → plan-store `[gap]` goal routing when continuity is on; the §2.4 provenance-stamping capture path; `wireContinuity`/`wireWiki` granular entry points; learning-skill rendering with `{{domain}}`/`{{curriculum}}`/`{{sources}}` substitution and the `run_exam` tool over an injected `ExamRunner`; Thredz goal mirroring (spec scope only, local write authoritative, skip-and-warn failures); `runWikiBackendConformance` — the cross-backend contract suite (PR 19).

Does not own: the stores themselves, the runtime loop seams it fills (`runtime-core`), the spec/IR lowering (`compiler`), or dream scheduling (`dream-engine`).

## Inputs and Outputs

Inputs: the IR-lowered memory/continuity/wiki/thredz/learning fragment + injected dependencies (catalog, McpHost, embedder, event append, exam runner).

Outputs: registered tools on the catalog; `RunChatLoopOptions` seam closures; the merged skills/commands set; a conformance-suite export for backend implementers.

## Dependency Notes

Depends on `@crewhaus/memory-store`, `@crewhaus/continuity-store`, `@crewhaus/tool-plan`, `@crewhaus/wiki-store`, `@crewhaus/tool-wiki`, `@crewhaus/tool-memory`, `@crewhaus/default-skills`, `@crewhaus/skills-registry`, `@crewhaus/slash-commands`, `@crewhaus/infra-utils`. Byte-identity is pinned: specs without a `memory:` block compile to byte-identical bundles, and `continuity: false` restores pre-0.3.0 bytes exactly.

## First Implementation Slice

Shipped in factory PR 10 (target-cli + interpreter refactor pinned by byte-diff smoke tests); PR 11 threads the spec→IR fragment to all five shapes; PR 16 adds the thredz backend flip; PR 17 adds learning/exam wiring.

## Study References

`runtime-core`'s `opts.memory` seam (the inverted-DI precedent), MODULE-CATALOG-STATUS.md (the critical-path gap this closes); design doc §1 principle 1 and §10.
