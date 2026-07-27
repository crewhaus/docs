# dataset-registry

Status: implemented and tested
Dependency phase: 9 - Telemetry & eval
Catalog layer: R15 - Telemetry, Tracing, Eval
Origin in ordering: named in Part G; the lifecycle surface landed in the 0.4.x evals campaign
Workspace home: packages/dataset-registry
Targets: EVAL
Test layers: T1, T2

## Purpose

Dataset abstractions, splits, and golden sets — the versioned, split-aware store behind the `registry:<name>[@version][#split]` reference and the `crewhaus datasets` verb family. A dataset is a named record of immutable versions; each version carries its splits *and* the per-sample content hashes computed when it was written, so "did this dataset change?" has an answer rather than a guess.

## Boundaries

Owns: the record schema and its versioning (`put` auto-bumps, and overwriting an existing version is refused unless the new `PutOptions.allowOverwrite` is passed); deterministic split assignment; `verifySplitHashes(record)` with the `HashMismatch` type and `overallDatasetHash(record, splits)` — what `crewhaus datasets verify` gates CI on; the additive `releases` / `ReleaseEntry` history recording each deliberate `#test` holdout spend; and one hard provenance invariant, that a sample tagged `metadata.source: "synthetic"` carrying an `expected_output` is **refused** at `put` with a pointer at `synthetic_human_verified`.

Does not own: sample parsing and the loaders (`eval-dataset`); where the `metadata.source` taxonomy is *enforced* versus merely warned (registering out-of-taxonomy provenance warns at the CLI edge and never fails); the test-split lock's per-verb policy — which verbs may consume `#test`, and behind which flag, is `apps/cli`'s decision; the lint and audit passes over a record's contents (`apps/cli/src/dataset-lint.ts` / `dataset-audit.ts`).

## Inputs and Outputs

Inputs: `Sample[]` plus a split spec, or a resolved `registry:` reference.

Outputs: a stored version record with per-split sample hashes and provenance; resolved sample streams for a reference; hash-mismatch reports; release/burn history.

## Dependency Notes

Depends on `@crewhaus/eval-dataset` (sample shape) and `@crewhaus/infra-utils`. Deliberately offline — `verify`, `status`, `card` and the lint pass read the record and never call a model, so they are safe as CI gates.

## First Implementation Slice

Shipped well before 0.4.x. The evals campaign added the lifecycle half: `datasets verify|status|release|card` on the CLI side, and `verifySplitHashes` / `overallDatasetHash` / `PutOptions.allowOverwrite` / `releases` / the synthetic-never-gold invariant in the library.

## Study References

`helm/benchmark/scenarios/`; `lm-evaluation-harness/lm_eval/tasks/`; `dspy/.../datasets/`; `ragas/.../dataset.py`,`testset/`; dataset datasheets (Gebru et al.) for the `datasets card` shape.

Research focus: Split semantics; holdout hygiene.

## Validation Plan

Catalog tests: T1, T2. Primary risks: split leakage (a held-out sample reaching a training or tuning path) and silent record drift (a version's contents changing while its name stays put — the failure `verify` exists to catch).

Definition of done: tests are green, public types are exported from the intended workspace, failure modes use typed `CrewhausError`-style errors where applicable, and the catalog status can be updated without hand-waving.
