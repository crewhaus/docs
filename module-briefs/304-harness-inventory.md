# harness-inventory

Status: implemented and tested
Dependency phase: 18 - Deployment & studio
Catalog layer: F3 - Deployment & Operations
Origin in ordering: 0.5.0 harness manager (Hangar M1; factory PR #372) — the package extraction of the CLI's `fleet` core
Workspace home: packages/harness-inventory
Targets: All (cross-harness; not embedded in a bundle)
Test layers: T1, T3

## Purpose

Cross-harness discovery, the fleet inventory and health rollup, and the seam-injected bulk runner — the library core that used to live inside `apps/cli` as the `crewhaus fleet` implementation. Lifting it out is what let the Hangar console and `crewhaus fleet` answer the same questions the same way instead of growing two dialects of "is this harness healthy".

Everything is side-effect-free on import: discovery and aggregation are tolerant pure reads over harness directories, and the only side-effect surface — launching per-harness CLI invocations — is the injected `FleetRunner`.

## Boundaries

Owns:

- **`discoverHarnesses(root)`** — the walk for directories carrying a `crewhaus.yaml`, capped at depth 6 (a deep monorepo still resolves; a pathological tree cannot hang the scan) and never descending into `.crewhaus`, `node_modules`, `.git`, `dist`, or `.worktrees`. A directory that IS a harness is still descended into, because a demo tree may nest sub-harnesses; only its state dir is skipped. Results are sorted by directory for stable output.
- **`readSpecHeader` / `SpecHeader`** — the cheap partial read of a spec's `name:` and `target:` used everywhere a full parse would be waste (the registry's refresh, `crewhaus daemon`'s harness resolution, the inventory join). A header that cannot be read degrades rather than throwing.
- **`buildHarnessInventory` / `buildFleetInventory` / `buildHarnessHealth`** — the per-harness row and the fleet rollup: sessions, feedback, open incidents, last-eval health, the registry/manifest status, and the compiled-entry detection (`isCompiledEntryPath`).
- **`READ_ONLY_BULK_COMMANDS`** — the allow-list the bulk runner accepts without an explicit opt-in: `eval`, `doctor`, `security digest`, `audit verify`. Anything else is `mutating: true` and requires `--allow-mutating` plus per-harness confirmation through the injected `ConfirmMutating` seam.
- **`describeFleetExit`** — turning an exit code into a `FailureClass` and a human sentence, which is what lets a fleet board group by how harnesses died ("3 harnesses exited 31 — provider funding") rather than by raw number.
- **`FleetError`** — operational failures (missing root, empty filter, a disallowed bulk subcommand). The CLI entry file catches it and routes the message through `die()`; tests assert on `.message` without the process exiting.
- **The formatters** — `formatInventory`, `formatHealth`, `formatBulkReport`, `healthMark`, which are what keep the CLI's golden tests byte-stable across the extraction.

Does not own: the registry (`harness-registry` — this package walks a filesystem and does not know about `hrn_` ids), process supervision (`harness-supervisor`), the will-it-boot checks (`preflight`), or the `crewhaus fleet` verb surface itself, which is now a thin re-export in `apps/cli/src/fleet.ts`.

## Inputs and Outputs

Inputs: a root directory to walk, an optional glob filter, injected readers (`ManifestReader`, `EvalIndexReader`, `EvalHealthReader`) and an injected `FleetRunner` / `ConfirmMutating` for the bulk path.

Outputs: `DiscoveredHarness[]`, `HarnessInventory` rows, `HarnessHealth` rollups, `BulkCommandPlan` / `RunFleetBulkReport`, and their rendered text forms.

## Dependency Notes

Depends on `@crewhaus/errors` (exit codes and `FailureClass`), `@crewhaus/feedback-distill` (feedback counting through the same extractor the flywheel uses), and `@crewhaus/spec-registry` types. `@crewhaus/hangar-server` and `@crewhaus/harness-supervisor` both consume it, which is why it sits below them in the dependency order.

The lift out of the CLI was **byte-identical**: `apps/cli/src/fleet.ts` re-exports this package and the CLI's existing fleet tests run unmodified against it. No behavior change to `fleet list|status|run` came with the extraction.

## First Implementation Slice

Shipped in factory PR #372 as an extraction, alongside `@crewhaus/harness-registry` and `@crewhaus/preflight`; the `--group <name>` filter that joins fleet discovery to registry groups landed in the same PR.

## Study References

The `crewhaus fleet` implementation this was lifted from (its golden tests are the extraction's proof), and `harness-registry`, which answers the complementary question — `fleet` asks "what is under this directory right now", `harness` asks "what has this machine registered, wherever it lives".

## Validation Plan

Catalog tests: T1, T3. Primary risks: **a discovery walk that misses or duplicates a harness** (a nested harness inside a demo tree is the interesting case), and **a bulk run that mutates without asking** — which is why the allow-list is a closed map and everything off it is `mutating` by default.

Definition of done: the CLI's pre-existing fleet golden tests pass unmodified against the package, discovery is deterministic and depth-bounded, and a disallowed bulk subcommand names the allowed set in its refusal.
