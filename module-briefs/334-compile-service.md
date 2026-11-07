# compile-service

Status: planned
Dependency phase: 2 - IR & Compiler
Catalog layer: F2 - Compiler & Codegen
Origin in ordering: 0.9.0, the Home release ([#514](https://github.com/crewhaus/factory/issues/514))
Workspace home: packages/compile-service
Targets: All (it operates compiled harnesses; it is not embedded in one)
Test layers: T1, T3

## Purpose

Check and compile a draft spec without registering or writing anything. A person editing a draft in Home, the 0.9.0 name for Hangar's web console, should see every problem at once, and try a build, before anything touches a harness. So should `crewhaus lint`, the guided creation flow and the upgrade preview.

Four consumers need the compiler, every emitter and a process runner for a draft. The compiler itself must stay worker-safe, and preflight must not depend on it, so none of them can host this. It is a package of its own, heavy by design, and Home loads it on first use.

## Boundaries

Owns:

- **`checkSpec`.** Touches no filesystem, network or registry, and returns in one result every schema issue (or, once the schema passes, every cross-field issue), unknown keys, lowering errors, warnings marked informational or not, IR-pass findings, feature support, the credentials each model slot needs (names and presence only) and the schema that answered. A spec with an unknown key, or newer than the schema that answered, is marked read-only with the reason.
- **`compileDraft`.** Returns files, warnings and the bundle digest, and writes and registers nothing. `crewhaus compile` still registers, so the difference between a draft and a build stays visible.
- **`checkBuild`.** `compile --check` moved unchanged out of `apps/cli`: a temp directory, a credential-free environment, and every subprocess through an injected runner.
- **`previewUpgrade`.** Returns the migrations, this harness's support changes, the default-on notes that apply to this spec and no others, the files that change against the active build, and whether `compile --strict` still passes. The creation flow's upgrade view renders it, and its notes registry becomes the 0.9.0 row of `spec-changelog`'s upgrade-notes table.
- **Build identity.** `hashSpecSource` and `bundleDigest`, re-exported rather than kept as second copies. Brief 022's planned `./policy` entry on `@crewhaus/ir` holds them, the first under the name `specRevision`.
- **The shared lint pipeline.** `crewhaus lint` calls `checkSpec`, so there is one pipeline, and its JSON format adds `schema`, `issues` and `unknownKeys` beside `ok` and `findings`.
- **Which schema answered.** Every result names it, from `specSchemaIdentity()` in `@crewhaus/spec`. When the creation flow asks a harness's own pinned CrewHaus as well, that answer can only narrow what Home accepts.

The model walk it uses for the credential list, `enumerateModelSlots`, moves from `apps/cli/src/model-slots.ts` to the planned `./slots` entry on `@crewhaus/ir` and gains the slots it misses.

Does not own: the compiler and its emitters (`compiler-core`, brief 025; the `target-*` packages); parsing and the schema (`spec`); IR passes (`ir-passes`); the generated support manifest, the planned `./support` entry on `@crewhaus/ir`, whose answers it reports; guarded spec writes, the planned `./commit` entry on `@crewhaus/spec-patch`; kept builds, activation and gates (`control-service`, brief 325; `creation-engine`'s `./gates`, brief 337); and the upgrade flow's screens and verbs (brief 337).

## Inputs and Outputs

Inputs: spec source text as a string, with an optional environment of names for presence checks; for `checkBuild`, an injected process runner; for `previewUpgrade`, the harness's active build.

Outputs: plain data. `checkSpec` returns one result object, never a throw for anything a spec can contain. `compileDraft` returns in-memory files. Nothing here writes to a harness, registers a version or spends a credential value: credentials appear by name and presence only.

## Dependency Notes

Depends on `@crewhaus/compiler`, `@crewhaus/spec`, `@crewhaus/ir`, `@crewhaus/ir-passes`, `@crewhaus/preflight`, `@crewhaus/smoke-harness`, `@crewhaus/spec-changelog`, `@crewhaus/migration-engine`, `@crewhaus/tool-registry-manifest` and `@crewhaus/errors`. No new third-party dependency.

It carries the compiler and every emitter, so a bundle, the compiler, `runtime-core` and `control-records` never import it, and a guard test holds that. `preflight` stays free of it too: the dependency runs one way, from here to `preflight`.

## First Implementation Slice

`feat(compile-service): check and compile a draft spec without registering or writing anything` adds the package, after the model walk moves. Later PRs extend it: `previewUpgrade` arrives with the upgrade PR, `feat(creation): import read-only, adopt, preview and apply upgrades, and roll back to a kept build`. This brief lands in crewhaus/docs before the package does.

## Study References

`apps/cli/src/index.ts` (the `compile --check` path that moves unchanged); `apps/cli/src/model-slots.ts` (the walk); `apps/cli/src/bundle-manifest.ts` (`hashSpecSource`); `packages/smoke-harness` (compile-time checks per shape); `packages/spec-changelog` (the upgrade-notes table); and [#514](https://github.com/crewhaus/factory/issues/514).

## Validation Plan

Catalog tests: T1, T3. Primary risks: **a check that writes**: a test runs `checkSpec` and `compileDraft` with a read-only working directory and asserts no file and no registry row changes; **a heavy import reaching a bundle**: a guard test walks the dependency tree of `runtime-core`, the compiler and `control-records`; **a second pipeline drifting from the first**: `crewhaus lint` and `checkSpec` answer from one function; **a model slot the walk misses**: a fixture per shape plants a distinct model string in every model-bearing field, and the test fails on any the walk misses.

Definition of done: `bun test packages/compile-service/src` green; `crewhaus compile --check` behaves as it did before the move, its subprocesses reached through the injected runner; the `previewUpgrade` notes stay equal to the CHANGELOG's default-on list, held by `scripts/default-on-changes.test.ts`; and every issue a spec can have comes back in one result.
