# tool-wiki

Status: implemented and tested
Dependency phase: 11 - Built-in tool implementations
Catalog layer: R4 - Built-in Tool Implementations
Origin in ordering: v0.3.0 memory release (design §3.2, PR 9)
Workspace home: packages/tool-wiki
Targets: CLI, CHN, MGD, RES, CRW
Test layers: T1, T8

## Purpose

One wiki tool vocabulary for both backends: `wiki_recall`, `wiki_semantic_search`, `wiki_search`, `wiki_get`, `wiki_write`, `wiki_list`, `wiki_related`, `wiki_set_signals`, `wiki_stats`, `log_knowledge_gap` — exact thredz-mcp names and schemas, pinned by a parity test. When `thredz:` flips the backend (PR 16), these local twins are NOT registered; the synthesized server's tools register under the same bare names via tool-mcp aliases, so skills, permission rules, and `tool_config` entries carry over unchanged.

## Boundaries

Owns: the RegisteredTools over a WikiStore and their Pillar 3 flags — `wiki_write`/`wiki_set_signals` destructive + `requireJustification: true`; `log_knowledge_gap` sideEffect audit-and-allow (standalone it records gaps as draft articles under the reserved `gaps/` tag; an injected `logGap` callback reroutes it to the plan store in the composition root). With `requireSources: true` (stamped by the `learning:` lowering), `wiki_write` deterministically rejects bodies without a `## Sources` heading. Recalled bodies are classified at the new `memory` TrustOrigin (block tier, like `skill`) and lineage-tagged before reaching the model — this package is one of the doctor `--philosophy-alignment` boundary sites.

Does not own: storage (`wiki-store`), the thredz alias routing (`tool-mcp`'s `registerMcpToolAliases`), or auto-recall fusion (`memory-service`).

## Inputs and Outputs

Inputs: a WikiStore, boundary-classifier context, the injected event-append + `logGap` seams.

Outputs: registered tools; `wiki_write` event-kind emissions; classified, lineage-tagged recall content.

## Dependency Notes

Depends on `@crewhaus/tool-builder`, `@crewhaus/wiki-store`, `@crewhaus/boundary-classifier`, `@crewhaus/errors`. Registered by `memory-service`'s `wireWiki`.

## First Implementation Slice

Shipped in factory PR 9; the `memory` TrustOrigin registered across boundary-classifier / run-context / egress-classifier in the same PR.

## Study References

thredz-mcp v0.2.0 (the tool contract source of truth), `tool-mcp` (the alias twin), AGENTS.md Pillar 3 contributor rules; design doc §3.2.
