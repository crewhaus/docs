# wiki-store

Status: implemented and tested
Dependency phase: 8 - Context & memory
Catalog layer: R6 - Context & Memory
Origin in ordering: v0.3.0 memory release (design §3.1, PR 9)
Workspace home: packages/wiki-store
Targets: CLI, CHN, MGD, RES, CRW (via `memory.wiki`; implied by `learning:`)
Test layers: T1, T3

## Purpose

The update-in-place semantic memory tier the append-only fact store cannot be: markdown articles with YAML frontmatter (slug, title, tags, confidence, verified, version, sources, supersedes, createdBy) under `.crewhaus/wiki/<spec>/`, immutable prior versions under `versions/<slug>/<n>.md` (supersede, never delete), and a rebuildable `index.json` carrying `[[wikilink]]` link graphs.

Retrieval is the release's main quality lever, applied for real: hybrid BM25 + optional embedder via reciprocal-rank fusion over contextual chunks (title + tags prefixed), followed by one-hop link-graph expansion with a documented half-weight re-rank rule — a linked-but-lexically-unrelated article surfaces on recall.

## Boundaries

Owns: the article/version/index layout; `write()` upserts with the Thredz PATCH optimistic-concurrency contract (a stale `expectedVersion` throws a `stale_article_version`-coded conflict, so skills behave identically on both backends); hybrid recall + link expansion; staleness-first listing and quality signals; §7.6 locking (advisory `.lock`, tmp+rename atomic writes) and fail-closed tenant fencing.

Does not own: the tool vocabulary (`tool-wiki`), backend selection (`memory-service`'s `wireWiki` — `file` here, `thredz` over McpHost in PR 16), the `memory.wiki` spec block, or consolidation (`dream-engine`).

## Inputs and Outputs

Inputs: store root + spec name (+ tenant), article writes with expected versions, recall/search queries, an optional `@crewhaus/embedder` instance.

Outputs: `WikiHit[]` recall bundles, article reads with full frontmatter, upsert `{slug, version}` results, staleness/stats reports.

## Dependency Notes

Depends on `@crewhaus/errors`, `@crewhaus/infra-utils` (file lock), optionally `@crewhaus/embedder`. Enrolled in `memory-service`'s `runWikiBackendConformance` suite (PR 19) — "local and Thredz are contract-identical" is a test, not a convention.

## First Implementation Slice

Shipped in factory PR 9 with `tool-wiki`; hybrid-recall regression pins BM25-only byte-identical behavior when no embedder is passed.

## Study References

Thredz wiki API (the PATCH contract this mirrors), contextual-retrieval evidence (chunk headers), GraphRAG/HippoRAG (the one-hop expansion is the cheap version); design doc §3.1.
