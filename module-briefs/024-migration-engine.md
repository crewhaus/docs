# migration-engine

Status: implemented and tested
Dependency phase: 2 - IR & Compiler
Catalog layer: F1 - Spec & IR
Origin in ordering: named in Part G; the first real migration shipped with the v0.6.0 model-plan release (factory PR #433)
Workspace home: packages/migration-engine
Targets: All
Test layers: T1, T4

## Purpose

Versioned **spec-schema** migrations — the upgrade path for a spec whose `version:` predates the CLI reading it. Migrations register as `{ from, to, up, down }` and the engine walks the chain in either direction, so a rollback is as expressible as an upgrade.

Note the unit: this migrates the spec schema, not the IR. The IR is a compile-time artefact rebuilt from the spec on every compile and has nothing to migrate.

Through v0.5.x the registry held only the no-op skeleton 0 → 1. v0.6.0 added the first real step, 1 → 2, which stamps the version and makes one default visible (`reward: { quality_source: none }`, and only when a `model_pool` with `policy: learned` exists). It is deliberately small — a version stamp plus an explicit default is trivially reversible.

## Boundaries

Owns:

- **The registry and the chain walk.** Forward when `from < to`, reverse when `from > to`, bridging any pair of versions through the registered steps.
- **`up(spec)`** — the object-level truth, a pure transform for callers that hold only a parsed object.
- **`edits?(spec)`** — the same change declared as a list of path/value edits, for callers that hold the YAML **text**. `crewhaus upgrade` and `migrate-all` apply those through the CST writer, so comments and key order survive; a step that supplies no edit list falls back to the object-level path and re-serialises, and the plan says per step which happened.
- **`irreversible?: true`** — the marker for a genuinely lossy step. The down-walk throws `MigrationIrreversibleError` across such a step instead of calling its `down()`, and the fleet runner surfaces that as a plan item rather than skipping it. The 1 → 2 step is not marked.

Does not own YAML serialisation (callers round-trip through the spec parser or the CST writer), spec validation (each migrated spec is re-validated through `parseSpec` before it can be written), or the memory-entry migration, which has a real inverse and lives with the memory CLI.

## Inputs and Outputs

Inputs: a parsed spec object and a target version; for the edit seam, the same spec plus the caller's own YAML text.

Outputs: the migrated spec object, a per-step plan naming the edits (or their absence), and typed `MigrationError` / `MigrationIrreversibleError` failures.

## Dependency Notes

Depends on `@crewhaus/errors` alone — the engine is serialisation-free, which is what lets both the CST writer and a plain object caller drive it. `MigrationEdit` is structurally identical to `@crewhaus/spec-patch`'s `SpecEdit` and declared here so that one dependency stays the whole footprint.

An older CLI does **not** refuse a newer spec: the `version` field is an optional non-negative int, so a 0.5.x CLI parses and compiles a v2 spec with 0.5.x semantics. Only `crewhaus upgrade` reports the drift, in both directions.

## First Implementation Slice

Shipped with the deployment section as the engine plus the 0 → 1 no-op. Factory PR #433 added the first real migration, the edit seam, the reachable irreversible guard, and comment-preserving `upgrade` / `migrate-all`.

## Study References

`claude-code/.../migrations/` (`CURRENT_MIGRATION_VERSION = 11`); LangGraph checkpoint version migrations

Research focus: Forward + backward migration paths; deprecation policy

## Validation Plan

Catalog tests: T1, T4. Primary risks: **a migration that destroys authorship** — comments and key order flattened by a re-serialise, which the edit seam closes — and **an unreachable guard**, a marker that never fires because no CLI path walks down.

Definition of done: tests are green, a migrated spec re-validates before it is written, a comment-bearing spec survives the upgrade byte-for-byte outside the edited paths, and the irreversible guard is exercised from a real CLI path.

