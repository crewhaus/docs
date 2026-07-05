# model-router

Status: implemented and tested
Dependency phase: 3 - Model & Tool primitives
Catalog layer: R2 - Model Layer
Origin in ordering: named in Part G
Workspace home: packages/model-router
Targets: All
Test layers: T1, T2, T7

## Purpose

Parse `agent.model` strings and lazy-load the matching `ProviderAdapter`. Every model call in a compiled harness routes through `resolveModel(modelString)`.

The router is the single owner of the model-string grammar: `claude-*` (unprefixed, Anthropic), `openai/<model>`, `gemini/<model>`, `bedrock/<modelId>` (family inferred from the id, cross-region inference-profile prefixes tolerated), `local/<model>[@<url>]` (any OpenAI-compatible server; defaults to Ollama's `http://localhost:11434/v1`), eight named OpenAI-compatible cloud hosts (`groq/`, `together/`, `fireworks/`, `openrouter/`, `deepseek/`, `xai/`, `mistral/`, `cerebras/`), `azure/<deployment>`, and `vertex/claude-*` / `vertex/gemini-*`. The user-facing matrix lives in [PROVIDERS.md](../PROVIDERS.md).

Since 0.2.0 the package also owns the two spec-native routing layers built on that resolution path: `createFailoverChain` — the breaker-driven failover meta-adapter behind `agent.model_fallbacks` / `agent.circuit_breaker` ([#264](https://github.com/crewhaus/factory/pull/264)) — and `createTierRouter` / `pickTier` — the deterministic two-tier turn-difficulty router behind `agent.model_tiers` ([#268](https://github.com/crewhaus/factory/pull/268)).

## Boundaries

Owns `parseModelString` (model string → discriminated union), `resolveModel` (parsed string → `{ adapter, modelId, providerId }`), the per-`(provider, baseUrl/deployment/family, key-env)` adapter cache, and the API-key policy for the OpenAI-routed paths — named hosts read their own key env var (never `OPENAI_API_KEY`); `local/` loopback URLs may inherit `OPENAI_API_KEY`, while non-loopback URLs only ever receive `CREWHAUS_LOCAL_API_KEY` so a spec-supplied URL cannot exfiltrate the OpenAI key.

Also owns the failover chain (`failover.ts`: per-candidate breaker wrapping, `model_failover` events with reasons `breaker_open` / `probe_restore` / `candidate_error`, tolerant fallback preflight surfaced via `warnings()`, `tripActive()` backing the `switch-model` recovery action, the `rankFallbacks` reordering seam, `FailoverExhaustedError`) and the tier router (`tier-router.ts`: deterministic `pickTier` over per-turn signals, boot-resolved `fast`/`default` tiers, `escalation()` for fast-tier misroute recovery).

Does not own adapter behavior — `@crewhaus/adapter-openai`, `@crewhaus/adapter-gemini`, and `@crewhaus/adapter-bedrock` are optionalDependencies loaded with dynamic `import()` only when a model string routes to them (a missing install fails with a `ConfigError` naming the package). Cost/quality policy (market scan, right-sizing) lives in the CLI's model tooling atop `cost-tracker`, and rate limiting lives in `rate-limiter`; the breaker state machine itself stays in `circuit-breaker` — the failover chain composes `wrap()` per candidate and never re-implements the transition rules.

## Inputs and Outputs

Input is the spec's `agent.model` string plus the process env. Output is a `ModelResolution` — the lazily-constructed adapter, the wire `modelId`, and the `providerId`. Malformed or unrecognised strings reject at parse time with a `ConfigError` carrying the accepted-grammar hint. The failover chain additionally takes the ordered `agent.model_fallbacks` strings and `agent.circuit_breaker` tuning and returns a `FailoverChain` — a `ProviderAdapter` with routing introspection (`plan` / `lastServed` / `candidates` / `warnings` / `tripActive`); the tier router takes two boot-resolved tiers plus `model_tiers.routing` and returns per-turn `TierDecision`s.

## Dependency Notes

Depends on `@crewhaus/adapter-anthropic` (the `ProviderAdapter` interface, always installed), `@crewhaus/errors`, `@crewhaus/circuit-breaker` (per-candidate wrapping in the failover chain), and `@crewhaus/trace-event-bus` (the late-bound bus for `model_failover` / `circuit_state_changed` events). The Bedrock family table and geo-prefix regex are twinned with `adapter-bedrock/src/family.ts` — the router cannot import the optional package eagerly, so the two copies carry identical test vectors.

## First Implementation Slice

The repo has the full tested surface. The next slice should preserve the existing exported grammar — adding a provider means adding a prefix branch in `parse.ts`, a lazy-import seam in `router.ts`, and updating the README table plus docs/PROVIDERS.md in the same change.

## Study References

`packages/model-router/README.md` (grammar table); `packages/model-router/src/parse.ts`; `packages/model-router/src/router.ts`; `packages/model-router/src/failover.ts`; `packages/model-router/src/tier-router.ts`

Research focus: grammar extensions; key-isolation policy for new host classes; failover/tier routing semantics

## Validation Plan

Catalog tests: T1, T2, T7. Primary risks: contract drift with providers or protocols; the twin Bedrock family tables in router and adapter drifting apart.

Definition of done: tests are green, public types are exported from the intended workspace, failure modes use typed `CrewhausError`-style errors where applicable, and the catalog status can be updated without hand-waving.
