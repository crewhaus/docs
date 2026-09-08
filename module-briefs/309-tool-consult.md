# tool-consult

Status: implemented and tested
Dependency phase: 11 - Built-in tool implementations
Catalog layer: R4 - Built-in Tool Implementations
Origin in ordering: v0.6.0 model-plan release (design §7.2.4/§7.5; factory PR #432)
Workspace home: packages/tool-consult
Targets: All pool-bearing shapes except RAG
Test layers: T1

## Purpose

The two model-directed hybrid tools. `Consult({ question, context?, to? })` lets the serving model ask a roster sibling one question and get one answer back; `Escalate({ reason })` lets it admit the turn is beyond it. Both are registered by `model-service` only when the pool declares `strategy.model_directed: true` — a tool-surface decision, so it stays human-owned.

The point is that a cheap worker can reach a stronger sibling without the operator having to predict in advance which turns will need one.

## Boundaries

Owns:

- **`Consult`** — the tool and its runner contract. The side call is a nested single-turn `runChatLoop` with no tools, never a raw `adapter.stream`, so `model_request` / `model_response`, cost accrual and budget metering all hold for the consulted model exactly as they do for the primary.
- **`Escalate`** — the receipt and the latch the loop consumes. It records a `model_stage` with `stage: "escalate"`, `cause: "self"`; `strategy.max_escalations` (default 1) bounds it, and past the cap the call records a skipped stage and returns a not-a-failure receipt rather than erroring.
- **The `consult` TrustOrigin** — registered across every site of the origin union, and the classification that goes with it. The reply re-enters the parent's context through one `classifyBoundary(..., { origin: "consult" })` pass followed by a lineage tag for the egress fabric; the tool sets `classifyOutput: false` so the runtime does not classify the same text twice, which Pillar 3 forbids.
- **Roster allowlist validation** — `Consult.to` is a model-filled argument, so it resolves *only* against `model_pool.candidates`: a profile name, a tag, or the exact candidate model string. Anything else returns an `is_error` tool result. Nothing model-filled can name a model outside the spec, and no adapter is ever resolved from model text at runtime.
- **`narrowRuleSet`**, the decision-level meet a per-profile `permissions` block needs — deny beats ask beats allow, proved through evaluation rather than by editing rule arrays.

Does not own: the runner (injected by `model-service`), the escalation *policy* (the loop decides what a captured latch does), or the strategy lanes — guide, shadow and committee are side calls, not tools.

## Inputs and Outputs

Inputs: an injected consult runner, the pool roster, the run context and bus, and the tool-call payloads.

Outputs: two `RegisteredTool`s for the catalog, an `EscalationLatch` the loop reads, and `model_stage` events for `route explain`, Hangar and OTel.

## Dependency Notes

Depends on `@crewhaus/tool-builder`, `@crewhaus/tool-catalog`, `@crewhaus/boundary-classifier`, `@crewhaus/run-context`, `@crewhaus/trace-event-bus` and `@crewhaus/errors`.

Permission posture follows the `Task` precedent: `readOnly`, `concurrencySafe`, `scope: "internal"`, no I/O capability — the socket is opened by the adapter layer inside the runtime's own metered loop, not by the tool. It deliberately never joins the builtin bookkeeping allow rules (whose contract is "no network, no process boundary"), so it takes the mode default in default and auto modes and is allowed in plan mode as a side-effect-free tool. An operator who wants it auto-allowed headless writes a spec permission rule, which outranks builtin.

## First Implementation Slice

Shipped in factory PR #432 with the TrustOrigin registration, `narrowRuleSet`, and interpreter/loop-contract parity, so the compiled bundle and `crewhaus run` register the identical pair.

## Study References

`tool-task` (the permission posture and the nested-run pattern), `sub-agent-spawner` (classify-once-at-the-boundary), and `model-service`'s side calls (the shared nested single-turn runner).

## Validation Plan

Catalog tests: T1. Primary risks: **a model naming its own model** — closed by resolving `to` against the roster only — and **double classification**, closed by the explicit `classifyOutput: false`.

Definition of done: tests are green, a consult failure or timeout returns an `is_error` result and the parent turn continues (the tool never throws past the executor), and an off-roster `to` is refused with a message naming the roster.
