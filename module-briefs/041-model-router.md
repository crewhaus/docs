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

## Boundaries

Owns `parseModelString` (model string → discriminated union), `resolveModel` (parsed string → `{ adapter, modelId, providerId }`), the per-`(provider, baseUrl/deployment/family, key-env)` adapter cache, and the API-key policy for the OpenAI-routed paths — named hosts read their own key env var (never `OPENAI_API_KEY`); `local/` loopback URLs may inherit `OPENAI_API_KEY`, while non-loopback URLs only ever receive `CREWHAUS_LOCAL_API_KEY` so a spec-supplied URL cannot exfiltrate the OpenAI key.

Does not own adapter behavior — `@crewhaus/adapter-openai`, `@crewhaus/adapter-gemini`, and `@crewhaus/adapter-bedrock` are optionalDependencies loaded with dynamic `import()` only when a model string routes to them (a missing install fails with a `ConfigError` naming the package). Policy routing (cost/quality/latency), failover, and rate limiting live in their own modules (`circuit-breaker`, `rate-limiter`).

## Inputs and Outputs

Input is the spec's `agent.model` string plus the process env. Output is a `ModelResolution` — the lazily-constructed adapter, the wire `modelId`, and the `providerId`. Malformed or unrecognised strings reject at parse time with a `ConfigError` carrying the accepted-grammar hint.

## Dependency Notes

Depends only on `@crewhaus/adapter-anthropic` (the `ProviderAdapter` interface, always installed) and `@crewhaus/errors`. The Bedrock family table and geo-prefix regex are twinned with `adapter-bedrock/src/family.ts` — the router cannot import the optional package eagerly, so the two copies carry identical test vectors.

## First Implementation Slice

The repo has the full tested surface. The next slice should preserve the existing exported grammar — adding a provider means adding a prefix branch in `parse.ts`, a lazy-import seam in `router.ts`, and updating the README table plus docs/PROVIDERS.md in the same change.

## Study References

`packages/model-router/README.md` (grammar table); `packages/model-router/src/parse.ts`; `packages/model-router/src/router.ts`

Research focus: grammar extensions; key-isolation policy for new host classes

## Validation Plan

Catalog tests: T1, T2, T7. Primary risks: contract drift with providers or protocols; the twin Bedrock family tables in router and adapter drifting apart.

Definition of done: tests are green, public types are exported from the intended workspace, failure modes use typed `CrewhausError`-style errors where applicable, and the catalog status can be updated without hand-waving.
