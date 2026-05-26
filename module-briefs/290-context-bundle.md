# context-bundle

Status: implemented and tested
Dependency phase: 1 - Foundations
Catalog layer: R17 - Infrastructure & Cross-Cutting
Origin in ordering: Phase 0 of the MCP-server proposal (2026-Q2)
Workspace home: packages/context-bundle
Targets: All (used by the CLI; consumed by the cloud demo via sibling-path import)
Test layers: T1

## Purpose

Builds a single-markdown orientation manifest of CrewHaus's docs + recipes + spec schema, plus a small set of pure slicing helpers that the cloud demo's prompt builder reuses so the two surfaces don't drift.

The package owns the *parsing* of CrewHaus's documentation tree (slice the decision tree out of `walkthroughs/INDEX.md`, slice the discriminated-union excerpt out of `packages/spec/src/index.ts`, extract H1/H2/H3 headings from `MODULE-CATALOG.md`, walk `walkthroughs/` for filename + first-heading). The source files themselves remain the source of truth.

## Boundaries

Owns: `buildContextBundle({ docsRoot, demosRoot, factoryRoot })`, `discoverRoots()`, `sliceDecisionTree()`, `sliceSpecSchemaExcerpt()`. All pure functions; no global state.

Does not own: the runtime that exposes the bundle (that's `apps/cli`'s `context` subcommand). Does not own MCP transport, schema validation, or any compile-pipeline behavior — those belong to their existing packages.

## Inputs and Outputs

Inputs: file paths for a CrewHaus checkout (factory, docs, demos roots). `discoverRoots()` resolves these from env vars (`CREWHAUS_FACTORY_ROOT`, `CREWHAUS_DOCS_ROOT`, `CREWHAUS_DEMOS_ROOT`) or by walking upward from a start directory.

Outputs: a `{ markdown, sources }` pair. `markdown` is the bundled manifest; `sources` lists every file read with byte counts so callers can report freshness or diagnose missing inputs.

## Dependency Notes

Zero runtime dependencies. Pure Node `fs` + `path`. Lives in the foundation phase so it can be consumed by anything (CLI, cloud demo, future MCP target emitter) without cycles.

## First Implementation Slice

Implemented in Phase 0 of the MCP-server proposal as a low-risk validation step: ships docs access via `crewhaus context --bundle` to confirm whether the docs-half of the proposal is sufficient before committing to the full compiled MCP-server target (which would add an `IrMcpServerV0` variant, an emitter package, and an outbound-classifier policy).

If Phase 0 demand justifies it, the next step is `target: mcp-server` (a peer of `target: cli`) that consumes the same slicers via `handler.kind === "builtin"`.

## Study References

OpenAI's `llms.txt` convention; Anthropic's docs MCP server; the cloud demo's bespoke `loadIndex` / `loadSpecSchema` helpers (now subsumed by this package).
