# knowledge-index

Status: planned
Dependency phase: 14 - RAG layer
Catalog layer: R12 - RAG / Retrieval / Knowledge
Origin in ordering: 0.9.0, the Home release ([#520](https://github.com/crewhaus/factory/issues/520))
Workspace home: packages/knowledge-index
Targets: All (it operates compiled harnesses; it is not embedded in one)
Test layers: T1, T3, T8

## Purpose

One view of what a harness knows. A harness keeps knowledge in several stores: facts, a wiki, working memory, session summaries, lessons, ingested documents, datasets and, with Thredz, a hosted wiki and records. This package projects them into one item model and one rebuildable search index, for Home, the 0.9.0 name for Hangar's web console, and the CLI, and it holds the one door through which CrewHaus's own commands delete knowledge about a person.

It is a package of its own because the CLI, Home and people erasure all need the same model, `memory-service` is bundle-facing, and the stores stay ignorant of each other. Each store's library stays the only writer of its bytes. No model is called in indexing or search.

## Boundaries

Owns:

- **`KnowledgeItem`.** A projection with one adapter per store. Each item carries its lifecycle, author, origin, whether its evidence is still available, confidence only where the store has one, usage, freshness, and its operations as `{supported, reason, cliTwin | null}`. Its id is `<store>~<spec>~<partition>~<localId>[@<version>]`, a grammar `control-contract` owns. A suggestion appears as a proposal, never as a belief.
- **The index.** Rebuildable per-harness BM25 shards under `<hangarRoot>/cache/index/knowledge/` (mode 0600), ranked by the engine `tool-state` already has, through its planned `./bm25` export. The projector of brief 325 builds them off the read path, never on a read or at boot, and deleting them rebuilds the same answers. A read compares file sizes and times with the shard and re-checks only stale hits, writing nothing. A harness without a shard yet gets a bounded fold of names and says so.
- **Search.** Plain words plus a deterministic grammar (`type:fact`, `in:<harness>`, `#tag`, `since:7d`, `status:corrected`). No model interprets a query, and it is never stored. Results are filtered by who asks before anything is ranked, counted or cut. A work item shows state and kind, and its title and content only with `h.read.content`. Nothing about a harness the asker cannot read appears, not even a count.
- **What stays out.** Facts marked personal and per-rater preference files never enter a shard or a sync, nor do conversations that used a profile, and customer memory never appears in fleet search.
- **The usage fold.** Joins usage lines with run records and decision logs ("used in 12 runs", "used after it was corrected"), noting that it shows what was in a run's context, not why a decision was made. `lookup(refs)` lets the run inspector say that a memory a run used was corrected since.
- **`./erasure`, the one door.** The knowledge-store half of the one hard-delete path [#521](https://github.com/crewhaus/factory/issues/521) sanctions for a person's data. It refuses ordinary knowledge: it acts only on items marked personal, or on records `people-service` names for a subject, and anything else gets "This isn't marked as personal data. Forget or archive it instead." It runs a dry run first, listing what will be deleted and what cannot be. It is authorized, then typed: `h.retention.manage` on every harness whose copies the dry run lists, or the whole call is refused, plus the harness name typed to confirm; when `people-service` erases a person, its own check and typed name apply instead. Only then do the store purges run, each with a grant the door mints after its checks. The receipt holds no content, and the harness's digest key is rotated.

All other knowledge keeps supersede-never-delete, and the write covenant in `packages/hangar-server/src/m3.ts` changes once to name every sanctioned path that deletes.

Does not own: writing to any store, corrections and archives (`memory-store`, `wiki-store`, `continuity-store`, extended in place by [#520](https://github.com/crewhaus/factory/issues/520)); recording what a run used (`runtime-core`, `memory-service`); the `knowledge.*` operations over HTTP (`hangar-server`, brief 301); the authority to erase a person and the vault's crypto-shred (`people-service`, brief 329; `people`, brief 328); the `source-conflict` item's inbox (`attention`, brief 331); and the BM25 engine and agent recall's own ranking (`tool-state`).

## Inputs and Outputs

Inputs: a harness directory, read through each store's library and `tool-safety`'s `./fs`; usage lines under `.crewhaus/run/knowledge/usage/`; the asker's readable harnesses and content permission; a query string; for `./erasure`, a target and an authorization.

Outputs: items, pages with their position, hits with snippets only where `h.read.content` allows, usage summaries, a statement of what each answer covers, and an empty state that names its cause. Erasure returns a dry-run inventory and a content-free receipt.

## Dependency Notes

Depends on `@crewhaus/control-contract`, `@crewhaus/control-records`, `@crewhaus/memory-store`, `@crewhaus/wiki-store`, `@crewhaus/continuity-store`, `@crewhaus/session-store`, `@crewhaus/dataset-registry`, `@crewhaus/harness-lifecycle`, `@crewhaus/tool-safety`, `@crewhaus/infra-utils`, `@crewhaus/tool-work`'s `./work-items` (read only), `@crewhaus/eval-grader` (the criteria reader) and `@crewhaus/tool-state`'s planned `./bm25`. It never enters a compiled harness (a guard test), adds no spec key, IR field or TrustOrigin, and takes no third-party dependency.

## First Implementation Slice

`feat(knowledge-index): one item model over every store, a rebuildable search index, and the one erasure door` adds the package, once work items and criteria can be read. Later PRs extend it: the operations over `crewhaus.hangar.v1`, then Home's Knowledge section. This brief lands in crewhaus/docs before the package does.

## Study References

`packages/hangar-server/src/m3.ts` (the write covenant); `packages/hangar-server/src/omnibox.ts` (the command menu, which indexes names and leaves bodies out); `packages/memory-store/src/index.ts` and `packages/wiki-store` (the stores it projects); `packages/hangar-server/src/rollups.ts` (a disposable cache whose test holds that deleting it is safe); and [#520](https://github.com/crewhaus/factory/issues/520).

## Validation Plan

Catalog tests: T1, T3, T8. Primary risks: **a read that builds or writes**: a read sweep over a fixture fleet leaves every file byte-identical; **search leaking across harnesses**: results are filtered before counting, and a test asserts no count, facet or snippet from an unreadable harness; **a personal fact in a shard**: a planted fact marked personal appears in no shard and no sync; **the erasure door used on ordinary knowledge**: an unmarked item is refused, a harness the caller does not own refuses the whole call, and a dry run writes nothing.

Definition of done: `bun test packages/knowledge-index/src` green; deleting every shard rebuilds the same answers; every target of a purge resolves inside its harness before anything is unlinked; and a guard test keeps the package out of every bundle.
