# gateway-protocol

Status: implemented and tested
Dependency phase: 15 - Channels
Catalog layer: R16 - UI / TUI / Voice / Media
Origin in ordering: named in Part G
Workspace home: packages/gateway-protocol
Targets: CHN, MGD
Test layers: T1, T2

## Purpose

Wire protocol (REST routes, WS messages, JSON-RPC methods).

This module should be the single owner for that concern. Keep the boundary small enough that generated targets can compose it without inheriting unrelated runtime policy.

## Boundaries

Owns the behavior named above, its typed public contract, and the fixtures needed to prove it. It should expose boring, explicit APIs rather than requiring callers to know internal storage or provider details.

Does not own neighboring concerns from the same phase unless the catalog explicitly says so. If implementation pressure suggests merging modules, keep the module names visible as exported interfaces so the catalog can still map to code.

## The control client

Factory PR #509 moved the `crewhaus.control.v1` client here from `hangar-server`, as a new subpath, `./control-client`, beside `.`, `./control` and `./control-codegen`. It holds `createControlClient`, the lanes, port parsing and the refusal codes. It reads the token path from the protocol's own constants, so the package gained no dependency. Every head that drives a daemon shares it without depending on an HTTP server.

## Inputs and Outputs

Inputs are UI events, audio/media streams, gateway messages, or notifications. Outputs are rendered state, protocol messages, call events, and media artifacts.

## Dependency Notes

Build this after the stable contracts from phases 1 through 14. Within phase 15, earlier numeric briefs are the preferred stabilization order, but independent modules can still be developed in parallel if their write sets do not overlap.

## First Implementation Slice

Start with a minimal typed interface, an in-memory or no-op implementation where possible, and focused tests for the catalog responsibility. Wire it to one nearby consumer only after the contract is stable.

A good first PR should include package metadata, the public entrypoint, unit tests for happy and failure paths, and one README or doc comment showing how the next layer imports it.

## Study References

`openclaw/.../gateway/protocol/`; OpenAI App Server protocol

Research focus: Versioning

## Validation Plan

Catalog tests: T1, T2. Primary risks: contract drift with providers or protocols.

Definition of done: tests are green, public types are exported from the intended workspace, failure modes use typed `CrewhausError`-style errors where applicable, and the catalog status can be updated without hand-waving.

