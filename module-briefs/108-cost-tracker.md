# cost-tracker

Status: implemented and tested
Dependency phase: 9 - Telemetry & eval
Catalog layer: R2 - Model Layer
Origin in ordering: named in Part G
Workspace home: packages/cost-tracker
Targets: All
Test layers: T1, T7

## Purpose

Estimate $/run, attribute to tenant/agent/tool.

This module should be the single owner for that concern. Keep the boundary small enough that generated targets can compose it without inheriting unrelated runtime policy.

## Boundaries

Owns the behavior named above, its typed public contract, and the fixtures needed to prove it. It should expose boring, explicit APIs rather than requiring callers to know internal storage or provider details.

Does not own neighboring concerns from the same phase unless the catalog explicitly says so. If implementation pressure suggests merging modules, keep the module names visible as exported interfaces so the catalog can still map to code.

## The spend meter (`./spend`)

Status: planned

An in-process meter that makes a budget hold for runs that spend at the same time. Every model call reserves its worst case before dispatch, settles on the response and releases on failure. A call is admitted while spend so far plus other in-flight reservations stays under the cap, so a run that makes one call at a time decides as it did before. Sub-agents spend from the parent's meter as they go, with a nested cap.

The worst-case estimates move here from eval-optimizer-orchestrator's budget gate (`packages/eval-optimizer-orchestrator/src/budget-gate.ts`), which re-exports them. The period ledger is an injected port (brief 324's `./budget`), so this package stays filesystem-free: the worker-safe compiler imports it (`packages/compiler/src/index.ts`). No new dependency.

A call on a model nothing can price is "cost unknown" in every total, never `$0`, and a declared price, a declared `$0` included, makes it known.

It lands with `feat(cost-tracker): spend is never a guessed $0: unknown when unpriced, reserved before each call, budgeted by period` ([#516](https://github.com/crewhaus/factory/issues/516)).

## Inputs and Outputs

Inputs are normalized messages, tool schemas, model policy, credentials, and budget constraints. Outputs are model responses, usage data, and routing decisions.

## Dependency Notes

Build this after the stable contracts from phases 1 through 8. Within phase 9, earlier numeric briefs are the preferred stabilization order, but independent modules can still be developed in parallel if their write sets do not overlap.

## First Implementation Slice

Start with a minimal typed interface, an in-memory or no-op implementation where possible, and focused tests for the catalog responsibility. Wire it to one nearby consumer only after the contract is stable.

A good first PR should include package metadata, the public entrypoint, unit tests for happy and failure paths, and one README or doc comment showing how the next layer imports it.

## Study References

`claude-code/.../cost-tracker.ts`; `ragas/.../cost.py`

Research focus: Provider price tables; granular attribution

## Validation Plan

Catalog tests: T1, T7. Primary risks: load, long-run behavior, and storage pressure.

Definition of done: tests are green, public types are exported from the intended workspace, failure modes use typed `CrewhausError`-style errors where applicable, and the catalog status can be updated without hand-waving.

