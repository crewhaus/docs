# model-plan

Status: implemented and tested
Dependency phase: 3 - Model & Tool primitives
Catalog layer: R2 - Model Layer
Origin in ordering: v0.6.0 model-plan release (design §4.4/§5/§7; factory PR #425)
Workspace home: packages/model-plan
Targets: All
Test layers: T1

## Purpose

The pure arithmetic of "which model, with which settings, with which tools, for this call". A `models:` profile is declared once in the spec and referenced as `$name` from any model slot; everything needed to turn that declaration into a concrete request lives here, and nothing else does.

Pure and fs-free by construction: no filesystem, no network, no adapter construction, no clock beyond an injected one. That is what lets four very different consumers share one implementation — the compiler validates with it, runtime-core builds its per-candidate plan table with it, the CLI's `models` and `route` verbs report from it, and the Hangar server reads through it.

## Boundaries

Owns:

- **Profile references** — `resolveProfileRef`, `applyProfileDefaults`, `PROFILE_NAME_RE`. A `$ref` is a lower-time macro: slot-local keys override the profile field by field, and nothing downstream of the compiler ever sees a `$`.
- **Request params** — `buildRequestParams(profile, base)`: the max-tokens / thinking / effort / temperature triple a candidate's call carries, clamped against what the candidate can serve.
- **Subset-only advertisement** — `buildAdvertisement`, `unmetRequirement`, `toolConfigFor`. A profile's `tools:` may only ever *narrow* the block's list. Additive is not expressible, which is the security property, not a limitation.
- **Rule-directed routing** — `evaluateRules`, `validateRuleRegex`, `deriveSignalRecord`: the deterministic per-turn signals a `model_pool.rules` entry matches on.
- **User-directed routing** — `parseModelDirective`, the `/model …` grammar a typed input seam hands over.
- **Eligibility** — `eligibleCandidates`, `cheapestEligible`, `strongestArm`: which candidates can serve *this* turn given required features, an open breaker and a budget hint.
- **Fingerprints** — `planFingerprint`, `profileFingerprint`, `canonicalJson`. A settings change must change the fingerprint, or a learned policy keeps exploiting an arm that no longer exists.
- **Priors and the floor** — `loadPriors` / `seededScoreLookup` (eval-measured priors, pseudo-count capped), and `checkFloor` / `wilsonLowerBound` (the conservative bound below which a learned policy may not exploit an arm).

Does not own: adapters, stores, or any wiring. Constructing chains and breakers, opening the scoreboard and building closures is `model-service`'s job; selecting a plan per model call is `runtime-core`'s.

## Inputs and Outputs

Inputs: a lowered `IrModelProfile` (or the pool's candidate array), a base request, the turn's signals, and — for the priors and floor helpers — arm statistics a caller has already read.

Outputs: resolved profiles, request-parameter records, advertisement subsets with the reason a tool was withheld, rule verdicts, eligibility sets, stable fingerprints, and pass/fail floor verdicts.

## Dependency Notes

Depends on `@crewhaus/adapter-anthropic` for the provider request and feature types alone — no client, no credentials, nothing that opens a socket. Everything above it (compiler, runtime-core, CLI, hangar-server) depends on this package rather than on each other's copies of the same arithmetic, which is the whole point of extracting it.

## First Implementation Slice

Shipped in factory PR #425 alongside the roster-first `strongest` sentinel, `ToolDefinition.requiresModelFeatures`, the capability table's context-window and max-output columns, and the optional `ProviderAdapter.effectiveParams?()` projection every in-tree adapter implements.

## Study References

`model-router`'s policy router (the arm identity and escalation conventions this mirrors), `cost-tracker`'s capability and pricing tables (the source `eligibleCandidates` consults), and `permission-engine`'s first-match-wins evaluation — the reason a profile's `permissions` narrow decisions rather than edit rule arrays.

## Validation Plan

Catalog tests: T1. Primary risk: a plan that says one thing and a request that sends another. Every derivation is therefore a pure function with its own test file, and the advertisement's subset property is asserted rather than assumed.

Definition of done: tests are green, the package imports nothing that touches disk or network, `buildAdvertisement` can never return a tool the block did not declare, and a change to any profile field changes `planFingerprint`.
