# Model Providers

> The canonical reference for the `model:` field. Every spec's model
> string is parsed by the
> [`model-router`](https://github.com/crewhaus/factory/tree/main/packages/model-router)
> using a strict prefix grammar, and the matching provider adapter is
> lazy-loaded — adapters for providers you don't use are never imported,
> let alone constructed.
>
> This page is the user-facing matrix: which strings exist, which env
> vars each one reads, and what each route can do. The
> contributor-facing twin (adapter caching, optional-dependency seams)
> is [`packages/model-router/README.md`](https://github.com/crewhaus/factory/blob/main/packages/model-router/README.md).

---

## Table of contents

1. [The grammar at a glance](#the-grammar-at-a-glance)
2. [Direct APIs: Anthropic, OpenAI, Gemini](#direct-apis-anthropic-openai-gemini)
3. [Local servers: Ollama, vLLM, llama.cpp, LM Studio](#local-servers-ollama-vllm-llamacpp-lm-studio)
4. [OpenAI-compatible clouds](#openai-compatible-clouds)
5. [AWS Bedrock](#aws-bedrock)
6. [Azure OpenAI](#azure-openai)
7. [Google Vertex AI](#google-vertex-ai)
8. [Capabilities per route](#capabilities-per-route)
9. [Troubleshooting](#troubleshooting)

---

## The grammar at a glance

| Model string | Where it goes | Env vars |
|---|---|---|
| `claude-sonnet-4-6` (unprefixed `claude-*`) | Anthropic API | `ANTHROPIC_AUTH_TOKEN` (Claude subscription, recommended) or `ANTHROPIC_API_KEY`; `ANTHROPIC_BASE_URL` optional for gateways/proxies |
| `openai/gpt-4o-mini` | OpenAI API — or any OpenAI-compatible endpoint via `OPENAI_BASE_URL` | `OPENAI_API_KEY` (or `OPENAI_BASE_URL` alone for keyless endpoints) |
| `gemini/gemini-2.5-flash` | Gemini API — or Vertex AI when the Vertex env flags are set | `GEMINI_API_KEY` / `GOOGLE_API_KEY`; or ADC + `GOOGLE_CLOUD_PROJECT` (+ `GOOGLE_CLOUD_LOCATION`) for Vertex |
| `bedrock/us.anthropic.claude-sonnet-4-5-20250929-v1:0` | AWS Bedrock — family inferred from the model id, [inference-profile prefixes](#inference-profiles-the-us-prefix-is-not-optional) (`us.` / `eu.` / `apac.` / …) supported and required by AWS for current models | AWS credential chain or `AWS_BEARER_TOKEN_BEDROCK`; region from `AWS_REGION`/`AWS_DEFAULT_REGION` or your AWS profile |
| `local/llama3.2@http://localhost:11434/v1` | Any OpenAI-compatible server — the URL **must include `/v1`** | None for loopback servers (may inherit `OPENAI_API_KEY`); remote URLs only ever get `CREWHAUS_LOCAL_API_KEY` |
| `local/llama3.2` | Shorthand for the Ollama default (`http://localhost:11434/v1`) | None |
| `groq/…` `together/…` `fireworks/…` `openrouter/…` `deepseek/…` `xai/…` `mistral/…` `cerebras/…` | The named host's OpenAI-compatible API | The host's own key env (`GROQ_API_KEY`, `XAI_API_KEY`, …) — never `OPENAI_API_KEY` |
| `azure/<deployment>` | Azure OpenAI (your deployment name, not a model name) | `AZURE_OPENAI_ENDPOINT`, `AZURE_OPENAI_API_KEY`, optional `AZURE_OPENAI_API_VERSION` (default `2024-10-21`) |
| `vertex/claude-sonnet-4-6` | Claude on Google Vertex AI | ADC + `ANTHROPIC_VERTEX_PROJECT_ID` (or `GOOGLE_CLOUD_PROJECT`); region via `CLOUD_ML_REGION`/`GOOGLE_CLOUD_LOCATION` (default `us-east5`) |
| `vertex/gemini-2.5-flash` | Gemini on Vertex AI (Vertex mode forced) | ADC + `GOOGLE_CLOUD_PROJECT` (+ `GOOGLE_CLOUD_LOCATION`, default `us-central1`) |

Anything else is rejected with a `ConfigError` that lists the accepted
prefixes — a typo in `model:` fails fast with a hint, not deep inside
an SDK.

Credentials are read from the environment at run time; `bun run`
auto-loads `.env`, and
[`factory/.env.example`](https://github.com/crewhaus/factory/blob/main/.env.example)
has a commented block per provider. You only need credentials for the
prefixes your spec actually references.

---

## Direct APIs: Anthropic, OpenAI, Gemini

### Anthropic (`claude-*`, no prefix)

The default and most complete route — tool use, vision, extended
thinking, explicit prompt caching, and server-side web search.

```yaml
agent:
  model: claude-sonnet-4-6
```

| Env var | Role |
|---|---|
| `ANTHROPIC_AUTH_TOKEN` | OAuth token from a Claude Pro/Max subscription (`claude setup-token`). Takes precedence when both are set. |
| `ANTHROPIC_API_KEY` | Pay-per-token API key from console.anthropic.com. |
| `ANTHROPIC_BASE_URL` | Optional — point the SDK at a gateway or proxy. |

### OpenAI (`openai/<model>`)

```yaml
agent:
  model: openai/gpt-4o-mini
```

`OPENAI_API_KEY` is the only required var. `OPENAI_BASE_URL` redirects
the client to any OpenAI-compatible endpoint (a corporate proxy, a
LiteLLM gateway, …); when a base URL is set, the key may be omitted
entirely — a placeholder is sent for servers that accept any key.

### Google Gemini (`gemini/<model>`)

```yaml
agent:
  model: gemini/gemini-2.5-flash
```

Two auth modes, resolved from the environment:

| Mode | Env vars |
|---|---|
| Gemini API (default) | `GEMINI_API_KEY` (also accepted as `GOOGLE_API_KEY`) |
| Vertex AI | `GOOGLE_GENAI_USE_VERTEXAI=true` forces it; it is also inferred when no API key is set but both `GOOGLE_CLOUD_PROJECT` and `GOOGLE_CLOUD_LOCATION` are. No API key — auth flows through Application Default Credentials. `GOOGLE_CLOUD_LOCATION` defaults to `us-central1`. |

So the same `gemini/…` spec runs against the consumer Gemini API on
your laptop and against Vertex AI in a GCP deployment, switched purely
by environment. (To *always* force Vertex regardless of env, use the
[`vertex/gemini-*` string](#google-vertex-ai) instead.)

---

## Local servers: Ollama, vLLM, llama.cpp, LM Studio

The `local/` grammar reuses the OpenAI adapter against any
OpenAI-compatible server. Two forms:

- `local/<model>` — shorthand for Ollama's default endpoint
  (`http://localhost:11434/v1`).
- `local/<model>@<url>` — explicit URL. **The URL must include the
  `/v1` segment** — the client appends `/chat/completions` to whatever
  you give it, so omitting `/v1` produces a 404 from the server.

| Server | Default endpoint | Example model string |
|---|---|---|
| Ollama | `http://localhost:11434/v1` | `local/llama3.2` (shorthand covers it) |
| vLLM | `http://localhost:8000/v1` | `local/meta-llama/Llama-3.1-8B-Instruct@http://localhost:8000/v1` |
| llama.cpp server | `http://localhost:8080/v1` | `local/qwen2.5-coder@http://localhost:8080/v1` |
| LM Studio | `http://localhost:1234/v1` | `local/llama-3.2-3b-instruct@http://localhost:1234/v1` |

Worked spec — Ollama, nothing to configure beyond `ollama pull`:

```yaml
# crewhaus.yaml
name: local-helper
target: cli
agent:
  model: local/llama3.2
  instructions: |
    You are a concise local assistant.
```

Worked spec — vLLM on its default port:

```yaml
# crewhaus.yaml
name: vllm-helper
target: cli
agent:
  model: local/meta-llama/Llama-3.1-8B-Instruct@http://localhost:8000/v1
  instructions: |
    You are a concise local assistant.
```

**Key handling is deliberately asymmetric.** Loopback URLs
(`localhost`, `127.*`, `::1`, `0.0.0.0`) may inherit `OPENAI_API_KEY`
if it's set — that's the documented LiteLLM-on-localhost setup. Any
non-loopback URL only ever receives `CREWHAUS_LOCAL_API_KEY` (or a
placeholder when unset) — a URL embedded in a third-party spec cannot
exfiltrate your OpenAI key. If your "local" server actually lives on
another machine and needs auth, set `CREWHAUS_LOCAL_API_KEY`.

See [`walkthroughs/32-local-models.md`](https://github.com/crewhaus/demos/blob/main/walkthroughs/32-local-models.md)
for the full local-models recipe.

---

## OpenAI-compatible clouds

Eight hosts are addressable by name. Each prefix pins the host's
chat-completions base URL and reads **its own** key env var — never
`OPENAI_API_KEY` — so a spec can mix hosts (a Groq judge next to a
real-OpenAI agent) without the keys fighting over one variable.

| Prefix | Endpoint | Key env var |
|---|---|---|
| `groq/` | `https://api.groq.com/openai/v1` | `GROQ_API_KEY` |
| `together/` | `https://api.together.xyz/v1` | `TOGETHER_API_KEY` |
| `fireworks/` | `https://api.fireworks.ai/inference/v1` | `FIREWORKS_API_KEY` |
| `openrouter/` | `https://openrouter.ai/api/v1` | `OPENROUTER_API_KEY` |
| `deepseek/` | `https://api.deepseek.com/v1` | `DEEPSEEK_API_KEY` |
| `xai/` | `https://api.x.ai/v1` | `XAI_API_KEY` |
| `mistral/` | `https://api.mistral.ai/v1` | `MISTRAL_API_KEY` |
| `cerebras/` | `https://api.cerebras.ai/v1` | `CEREBRAS_API_KEY` |

The model id after the prefix is passed through verbatim, so hosts with
slashes in their model names work as-is:

```yaml
agent:
  model: together/meta-llama/Llama-3.3-70B-Instruct-Turbo
```

The host's key env var is **required** — a missing key fails
immediately with a `ProviderAuthError` naming the variable, rather than
a confusing 401 from the wrong endpoint.

**The escape hatch.** For an OpenAI-compatible cloud that isn't in the
table, use plain `openai/<model>` with `OPENAI_BASE_URL` pointing at
the endpoint and `OPENAI_API_KEY` carrying its key. That's the generic
form the named hosts are sugar for.

---

## AWS Bedrock

`bedrock/<modelId>` routes through `@crewhaus/adapter-bedrock`. The
family is inferred from the model id prefix; eleven families are
recognised:

| Model id prefix | Family |
|---|---|
| `anthropic.` | anthropic |
| `meta.llama` | llama |
| `mistral.` | mistral |
| `amazon.nova` | nova |
| `amazon.titan-text` | titan |
| `deepseek.` | deepseek |
| `cohere.command` | cohere |
| `ai21.` | ai21 |
| `qwen.` | qwen |
| `openai.gpt-oss` | gpt-oss |
| `writer.` | writer |

Prefixes are deliberately narrow where a vendor ships non-chat models
under the same segment (`amazon.titan-text` vs `amazon.titan-embed`,
`cohere.command` vs `cohere.embed`) so genuinely unsupported models are
rejected at parse time instead of failing mid-run.

### Inference profiles: the `us.` prefix is not optional

AWS invokes current-generation models through **cross-region inference
profiles** — the model id with a geo segment prepended (`us.` / `eu.` /
`apac.` / `jp.` / `au.` / `ca.` / `sa.` / `us-gov.` / `global.`). For
these models AWS *requires* the profile id; invoking the bare model id
returns a `ValidationException` saying on-demand throughput isn't
supported. So the model string you want is almost always the prefixed
form:

```yaml
agent:
  model: bedrock/us.anthropic.claude-sonnet-4-5-20250929-v1:0
```

The router strips the geo segment only to sniff the family — the full
profile id goes on the wire unchanged.

### Credentials and region

| Source | Notes |
|---|---|
| Standard AWS credential chain | `AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY` env vars, `~/.aws/credentials`, SSO profiles, IAM roles — whatever the AWS SDK resolves. |
| `AWS_BEARER_TOKEN_BEDROCK` | A Bedrock API key, read natively by the SDK — the simplest path if you're not otherwise an AWS user. |

Region comes from `AWS_REGION`/`AWS_DEFAULT_REGION` when set; otherwise
the client is built without an explicit region so the SDK's own chain
(including the region in your `~/.aws/config` profile) resolves it.
There is no hardcoded fallback region — a hardcoded default would
silently send profile-based users to a region where their model isn't
enabled.

### Two wire paths, one adapter

- **`anthropic.*` models stream over the native
  `InvokeModelWithResponseStream` path** with the Messages-API-shaped
  body — so explicit `cache_control` markers and the extended-thinking
  budget keep working exactly as they do against the Anthropic API.
- **Every other family goes through the model-agnostic
  Converse/ConverseStream API** — one request shape and one stream
  vocabulary regardless of vendor, which is what gives
  llama/mistral/nova/cohere/qwen/gpt-oss genuine tool use and
  deepseek/gpt-oss reasoning without per-vendor body marshalling.
  Converse has no client-controlled prompt caching, so caching is off
  for these families.

### Bedrock family capability matrix

| Family | Tool use | Vision | Thinking | Caching |
|---|---|---|---|---|
| anthropic | ✅ | ✅ | ✅ | explicit |
| llama | ✅ | ❌ | ❌ | ❌ |
| mistral | ✅ | ❌ | ❌ | ❌ |
| nova | ✅ | ✅ | ❌ | ❌ |
| cohere | ✅ | ❌ | ❌ | ❌ |
| qwen | ✅ | ❌ | ❌ | ❌ |
| gpt-oss | ✅ | ❌ | ✅ | ❌ |
| deepseek | ❌ | ❌ | ✅ | ❌ |
| titan | ❌ | ❌ | ❌ | ❌ |
| ai21 | ❌ | ❌ | ❌ | ❌ |
| writer | ❌ | ❌ | ❌ | ❌ |

The runtime reads these flags and degrades gracefully — e.g. specs
whose tools matter shouldn't pick a deepseek/titan/ai21/writer model on
Bedrock.

---

## Azure OpenAI

`azure/<deployment>` targets Azure OpenAI's classic surface
(deployment-scoped path, `api-key` header, `api-version` query). The
segment after the slash is **your deployment name** — the name you gave
the deployment in Azure AI Foundry — not an OpenAI model name.

```yaml
agent:
  model: azure/my-gpt4o-deployment
```

| Env var | Role |
|---|---|
| `AZURE_OPENAI_ENDPOINT` | `https://<resource>.openai.azure.com` — required. |
| `AZURE_OPENAI_API_KEY` | The resource key — required. |
| `AZURE_OPENAI_API_VERSION` | Optional; defaults to `2024-10-21` (a stable GA version). |

Once routing and auth are handled, the wire protocol is plain Chat
Completions — features match the OpenAI route.

---

## Google Vertex AI

Two model families run on Vertex, with different SDKs underneath:

### Claude on Vertex (`vertex/claude-*`)

```yaml
agent:
  model: vertex/claude-sonnet-4-6
```

Backed by `@anthropic-ai/vertex-sdk`, which is an **optional
dependency** — install it (`bun add @anthropic-ai/vertex-sdk`) before
first use; a missing install fails with a `ConfigError` naming the
package.

| Env var | Role |
|---|---|
| `ANTHROPIC_VERTEX_PROJECT_ID` (or `GOOGLE_CLOUD_PROJECT`) | Required — your GCP project id. |
| `CLOUD_ML_REGION` (or `GOOGLE_CLOUD_LOCATION`) | Optional; defaults to `us-east5`, the widest Claude availability on Vertex. |

Auth flows through Application Default Credentials (`gcloud auth
application-default login` locally, the ambient service account in
GCP) — no API key. Subscription (OAuth) billing does not apply on
Vertex.

### Gemini on Vertex (`vertex/gemini-*`)

```yaml
agent:
  model: vertex/gemini-2.5-flash
```

Forces the Gemini adapter into Vertex mode regardless of env flags
(`vertex/gemma-*` is accepted too). Requires `GOOGLE_CLOUD_PROJECT`;
`GOOGLE_CLOUD_LOCATION` defaults to `us-central1`; ADC for auth.

If you'd rather keep the spec portable and switch per environment, use
`gemini/<model>` with `GOOGLE_GENAI_USE_VERTEXAI=true` — see
[Direct APIs](#direct-apis-anthropic-openai-gemini).

---

## Capabilities per route

Each adapter declares a `features` matrix and the runtime routes around
it (e.g. prompt-cache rotation is skipped where caching is automatic or
absent). The four flags that matter when picking a model string:

| Route | Tool use | Vision | Thinking | Caching |
|---|---|---|---|---|
| `claude-*` (Anthropic direct) | ✅ | ✅ | ✅ | explicit (`cache_control`) |
| `vertex/claude-*` | ✅ | ✅ | ✅ | explicit (`cache_control`) |
| `openai/<model>` | ✅ | ✅ | ❌ | automatic (server-managed) |
| `azure/<deployment>` | ✅ | ✅ | ❌ | automatic (server-managed) |
| Named hosts (`groq/`, `xai/`, …) | ✅ | ✅ | ❌ | automatic (server-managed) |
| `local/<model>[@<url>]` | ✅ | ✅ | ❌ | automatic (server-managed) |
| `gemini/<model>`, `vertex/gemini-*` | ✅ | ✅ | ✅ | automatic (implicit caching) |
| `bedrock/<id>` | per family | per family | per family | per family — see the [matrix above](#bedrock-family-capability-matrix) |

Notes:

- "Automatic" caching means the API caches repeated prefixes
  server-side on its own; explicit `cache_control` markers are not sent
  on those routes.
- The OpenAI-adapter routes (`openai/`, `azure/`, named hosts,
  `local/`) advertise one shared matrix. What the *server* actually
  supports varies — a 3B model on Ollama may ignore tool definitions,
  and reasoning models behind OpenAI-compatible APIs handle thinking
  internally rather than via the canonical thinking blocks.
- Server-side web search is only advertised on the Anthropic API routes
  (`claude-*`, `vertex/claude-*`) — not on Anthropic-on-Bedrock.
  Everywhere else, give the agent the `WebSearch`/`WebFetch` tools
  instead.

---

## Troubleshooting

| Symptom | Cause and fix |
|---|---|
| `404` from a `local/` model string | The URL is missing the `/v1` segment. The client appends `/chat/completions`, so `local/llama3.2@http://localhost:11434` requests `/chat/completions` — which doesn't exist. Use `local/llama3.2@http://localhost:11434/v1` (or the bare `local/llama3.2` shorthand, which already includes it). |
| Bedrock `ValidationException` ("on-demand throughput isn't supported…") | You passed a bare model id. Current-generation models must be invoked through their cross-region inference profile — prepend the geo segment: `bedrock/us.anthropic.claude-sonnet-4-5-20250929-v1:0`, not `bedrock/anthropic.claude-…`. |
| `ProviderAuthError: groq/* model strings require GROQ_API_KEY to be set` | Named hosts read **only** their own key env var, never `OPENAI_API_KEY`. Set the variable the error names. |
| `ConfigError: … unrecognised model string` | Typo in the prefix. The error lists every accepted form — `claude-*`, `openai/*`, `gemini/*`, `bedrock/*`, `local/<model>[@<url>]`, `azure/<deployment>`, `vertex/<model>`, and the named hosts. |
| `ConfigError: @crewhaus/adapter-… is not installed` | The provider adapters are optional dependencies, loaded only when a model string routes to them. Install the named package. Same story for `@anthropic-ai/vertex-sdk` on `vertex/claude-*`. |
| `no Anthropic / OpenAI / Gemini credentials found` | Copy [`.env.example`](https://github.com/crewhaus/factory/blob/main/.env.example) to `.env` and set the variable(s) for your provider — `bun run` auto-loads `.env`. |
| Remote "local" server rejects the request as unauthenticated | Non-loopback `local/` URLs never receive `OPENAI_API_KEY` (deliberately — a spec-supplied URL must not be able to exfiltrate it). Set `CREWHAUS_LOCAL_API_KEY` with the remote server's key. |
| Bedrock call lands in the wrong region | Set `AWS_REGION` explicitly, or set the region on the AWS profile you're using — when neither env var is set, the SDK's chain (including `~/.aws/config`) decides. |

If the symptom isn't listed, the JSONL event log under
`.crewhaus/sessions/` is usually the fastest path to a diagnosis.
