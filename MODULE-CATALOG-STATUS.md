# Module Catalog — implementation status

This document tracks **what's built, what's in flight, and what's deferred** across the catalog. It complements [MODULE-CATALOG.md](MODULE-CATALOG.md), which describes the catalog's shape and is intended for navigation, not for ticket-tracking.

If you want the *what / why / how* of a module, go to [MODULE-CATALOG.md](MODULE-CATALOG.md) or [module-briefs/](module-briefs/README.md). If you want roadmap commitments and section-by-section history, go to [build-roadmap.md](https://github.com/crewhaus/operations/blob/main/build-roadmap.md). This document sits between them: a single-glance status board.

---

## Snapshot — 2026-05-10

- **Sections 1–40 shipped** — v1.0 + v1.1 + v1.2 product surface complete.
- **180 of ~190 catalog modules implemented** across 164 workspace packages + `apps/cli`.
- **12 target shapes ship today**: cli / workflow / channel / graph / managed / pipeline / crew / research / batch / voice / browser / eval.
- **2,526 tests across ~176 test files** plus a handful of live-probe contract tests gated on env vars; all green.
- `bun run tsc -b` clean. `biome check` clean. `example-corpus` CI matrix green.

The 10 unbuilt modules are all in v1.3+ and **not on any critical path**. See [v1.3 backlog](#v13-backlog) below.

---

## Status markers

Catalog tables and briefs use these markers:

| Marker | Meaning |
|---|---|
| ✅ | Implemented and tested (has `src/` + tests + at least one consumer). |
| 🚧 | In progress (open PR). |
| *(unmarked)* | Not started. |

Unbuilt modules carry a one-character risk marker placed before the module name:

| Risk | Meaning |
|---|---|
| 🔴 | Critical-path bottleneck (blocks ≥3 downstream modules or a target shape), security-sensitive surface, or substantive design unknowns. |
| 🟡 | Well-understood pattern but several non-trivial dependencies, novel-to-this-stack, or moderate blast radius if the contract changes. |
| *(no marker)* | Isolated, proven pattern with few dependents. |

Implemented (✅) and in-progress (🚧) modules have absorbed their risk and do not carry the marker. The marker is a guide for sequencing, not a blocker — a 🟡 module can ship in any phase as long as its `Depends on` row is satisfied.

---

## Implementation summary

Sections 1–40 are closed. The build-roadmap is the section-by-section authority; this list groups what each section delivered so a contributor can locate a module's history quickly.

| Sections | Phase | Delivered |
|---|---|---|
| §1–§5 | v1.0 — CLI core | spec / compiler / IR / runtime-core / tool framework / target-cli |
| §6 | v1.0 — workflow target | `target-workflow`, IrWorkflowV0, per-step model overrides |
| §7–§11 | v1.0 — safety + extension surface | permission-engine, hooks-engine, skills-registry, slash-commands |
| §12–§14 | v1.0 — channel + multi-modal | target-channel-bot, Slack adapter, web/image/fetch tools |
| §15 | v1.0 — observability | trace-event-bus, otel-exporter, metrics-collector, structured-event-printer |
| §16 | v1.0 — eval stack | eval-dataset, eval-grader, eval-judge, eval-runner, eval-report, `crewhaus eval` |
| §17 | v1.0 — multi-provider models | adapter-openai/gemini/bedrock, model-router with lazy SDK loading |
| §18 | v1.0 — safety floor | sandbox, tool-code-execution, prompt-injection-detector |
| §19 | v1.0 — graph target | target-graph, graph-engine, checkpoint-store, branch-history, durable-execution |
| §20 | v1.0 — managed target | target-managed, gateway-protocol/server, policy-engine, tenancy, audit-log |
| §21 | v1.0 — RAG target | target-pipeline, pipeline-engine, chunker, embedder, vector-store, tool-retrieve |
| §22 | v1.0 — crew target | target-crew, crew-orchestrator, agent-handoff, a2a-protocol |
| §23 | v1.0 — research + batch | target-research-bundle, target-batch-worker, queue-protocol, citation-tracker |
| §24 | v1.0 — voice target | target-voice, voice-runtime, vad-engine, barge-in-controller, call-session |
| §25 | v1.0 — browser target | target-browser-driver, computer-use-driver, tool-screen-capture/-mouse-keyboard/-vision-grounding |
| §26 | v1.0 — studio | wizard, scaffold-templates, plugin-sdk v0, studio-server, studio-ui, trace-viewer, graph-visualizer |
| §27 | v1.0 — production hardening | cost-tracker, secrets-manager, prompt-cache-manager, rate-limiter, circuit-breaker |
| §28 | v1.0 — deployment | spec-registry, ir-passes, migration-engine/runner, deployment-controller, canary-controller |
| §29 | v1.0 — eval depth | dataset-registry, grader-registry, regression-runner, prompt-optimizer, target-eval-bundle |
| §30 | v1.0 — backend adapters | SQS / Redis / Postgres queue adapters; Lance / Qdrant / Pinecone / Weaviate vector backends; Twilio / LiveKit telephony; Vapi realtime |
| §31 | v1.0 — studio v1 | Lit + Monaco + live SSE replay + plugin sandbox content isolation |
| §32 | v1.1 — distribution | docker-images, single-binary-cli, helm-chart, crewhaus-cloud (terraform + kustomize) |
| §33 | v1.1 — channel breadth | telegram / discord / whatsapp / imessage adapters |
| §34 | v1.1 — federation | federation-protocol / -discovery / -router (mTLS + sigstore-style pinning) |
| §35 | v1.1 — IDE & DX | vscode-extension, jetbrains-plugin, crewhaus-playground |
| §36 | v1.2 — polyglot sandboxes | go / rust / java / ruby / r / dotnet / php sandbox images on a registry pattern |
| §37 | v1.2 — vendor telemetry | datadog / honeycomb / splunk / newrelic exporters with credential-leak guards |
| §38 | v1.2 — production graders | NLG metrics, semantic similarity, safety classifiers, multimodal graders |
| §39 | v1.2 — compliance hardening | pii-redactor, audit-encryption, data-retention-engine, compliance-controls + `crewhaus compliance evidence` CLI |
| §40 | v1.2 — template marketplace | template-registry, template-marketplace-client, example-corpus CI matrix gate |

For per-section detail (file paths, kickoff prompts, PR refs), see [build-roadmap.md](https://github.com/crewhaus/operations/blob/main/build-roadmap.md).

---

## v1.3 backlog

The v1.2 phase closed out the breadth + ecosystem layer (§36–§40). v1.3 covers the remaining ~10 unbuilt catalog modules across:

- **§41 — plugin SDK v2 + plugin-loader** (committed, kickoff prompt ready)
- **§42 — plugin-registry + module-marketplace-client** (committed, depends on §41)
- **§43 — mobile target shapes** (deferred indefinitely; needs Bun-on-iOS / NDK to stabilise)
- **§44 — cloud-deploy adapters** (independent / opportunistic — Render / Fly.io / Railway / Heroku)
- **§45 — long-tail breadth** (no-section backlog; per-addition PRs)

| Module(s) | Layer | Section | Notes |
|---|---|---|---|
| `plugin-sdk` v2 (extend), `plugin-loader` (new) | F5 | §41 | Public typed surface for third-party tools/channels/models/graders/target-backends + runtime activation with sandboxed import + capability gating. Reuses §31 plugin-sandbox content-isolation + §40 sigstore-style Ed25519 signature verification + §29 grader-registry plugin discovery shape. **Sequential prereq for §42.** Unblocks the entire third-party ecosystem. |
| `plugin-registry` (new), `module-marketplace-client` (new) | F5 | §42 | Extend §40 template-marketplace pattern from templates to plugins. Browse + install community-published tools/skills/channels/graders/target-emitters directly from Studio. New `crewhaus plugins {list,search,install,uninstall}` CLI subcommands. |
| `target-ios-bundle`, `target-android-bundle` | F2 | §43 | Mobile target shapes. iOS Swift Package Manager bundle wrapping the Bun-on-iOS Hermes embedding pattern; Android AAR bundle wrapping the Bun-on-Android NDK bridge. **Deferred indefinitely** until (1) Bun publishes a stable iOS embedding API, (2) Android NDK bridge ships in Bun mainline, (3) at least one external partner signs an LOI, (4) CI runners with `xcodebuild` + `gradle` are provisioned. |
| `cloud-adapter-render`, `cloud-adapter-flyio`, `cloud-adapter-railway`, `cloud-adapter-heroku` | F3 | §44 | One-click cloud-deploy adapters for dev-friendly platforms. Each ~200 LOC of platform-specific deploy-API + target-specific Dockerfile/runtime config. All four parallelisable; independent of §41–43 and of each other. T8 credential-leak guard mirrors §37 vendor exporter pattern. (`cloud-adapter-vercel` deferred — depends on a `target-vercel-functions` shape not yet in the catalog.) |
| Long-tail breadth | various | §45 | Additional sub-agent templates, MCP servers, embedder backends (Cohere v3 / Mistral Embed / Voyage v3+), vector-store backends (Lance v0.10+, Postgres+pgvector), additional channel adapters (Microsoft Teams, Mattermost, Matrix), domain-specific graders. Each is small (typically <200 LOC) and piggy-backs on shipped infrastructure. Reference §45 in any per-addition PR title; promote to a dedicated section if a cluster (3+ related additions) emerges at once. |

None of v1.3 is critical-path for any shipped target shape.

---

## Critical path & risk register

Cross-references the per-layer `Depends on` columns in the catalog with the build phases. This section is the single place to look up "what is the marginal cost of unblocking shape X?" — every entry below is an unbuilt module whose absence would stall a target shape, a stack, or another high-leverage module.

> **Note (2026-05):** Sections 1–40 are closed. This register documents the historical critical path that has already been consumed. The entries marked ✅ shipped in the corresponding sections; they're retained here as a record of which dependencies the project navigated. The active critical path for v1.3+ work is the [v1.3 backlog](#v13-backlog) above.

### Historical high-risk (🔴) modules and their direct blockers (all now shipped)

| Module | Layer | Phase | Must land first | Unblocks (direct downstream) |
|---|---|---|---|---|
| `model-router` ✅ | R2 | 3 | `model-adapter`, `secrets-manager`, `auth-profiles` | `adapter-{openai,gemini,bedrock}`, multi-provider compaction, §17 |
| `target-graph` ✅ | F2 | 2 (gated by R11) | `compiler-core`, `codegen-templates`, `graph-engine`, `checkpoint-store` | GRPH shape, MGD partial |
| `target-pipeline` ✅ | F2 | 2 (gated by R11) | `compiler-core`, `codegen-templates`, `pipeline-engine` | RAG shape end-to-end |
| `target-managed` ✅ | F2 | 2 (gated by F3) | `compiler-core`, `bundle-packager`, `audit-log`, `tenancy` | MGD shape |
| `target-research-bundle` ✅ | F2 | 2 (gated by R19) | `compiler-core`, `research-planner`, `research-checkpointer`, `report-synthesizer` | RES shape |
| `target-voice` ✅ | F2 | 2 (gated by R16) | `compiler-core`, `voice-runtime`, `audio-stream`, `call-session` | VOICE shape |
| `target-browser-driver` ✅ | F2 | 2 (gated by R18) | `compiler-core`, `computer-use-driver`, `tool-screen-capture`, `tool-vision-grounding` | BROW shape |
| `deployment-controller` ✅ | F3 | 18 | `compiler-core`, `bundle-packager`, `secrets-manager` | `deployment-profiles`, `canary-controller`, `upgrade-controller`, MGD |
| `tool-code-execution` ✅ | R4 | 11 | `tool-builder`, `sandbox` | EVAL CI auto-run, RES auto-execution, MGD code samples |
| `tool-screen-capture` ✅ | R4 | 11 | `tool-builder`, `computer-use-driver` | BROW shape |
| `tool-mouse-keyboard` ✅ | R4 | 11 | `tool-builder`, `computer-use-driver` | BROW shape |
| `tool-vision-grounding` ✅ | R4 | 11 | `tool-builder`, `tool-screen-capture`, `model-adapter` | BROW grounded actions |
| `a2a-protocol` ✅ | R5 | 10 | `tool-catalog`, `model-adapter`, `webhook-host` | CRW cross-org, MGD federated agents |
| `checkpoint-store` ✅ | R7 | 6 | `event-log`, `checkpoint-encoder`, `state-store` | `branch-history`, `target-graph`, `durable-execution`, `research-checkpointer` |
| `branch-history` ✅ | R7 | 6 | `checkpoint-store`, `event-log` | GRPH time-travel, RES branch exploration |
| `policy-engine` ✅ | R8 | 7 | `permission-engine`, `audit-log`, `tool-permission-matcher` | `tenancy`, regulated MGD, `sandbox` compose |
| `sandbox` ✅ | R8 | 7 | `infra-utils`, `policy-engine` | `tool-code-execution`, `tool-browser`, `plugin-loader`, EVAL/MGD hardening |
| `prompt-injection-detector` ✅ | R8 | 7 | `tool-result-store`, `model-adapter` | `guardrails` depth, MGD audit, RES untrusted-source ingest |
| `graph-engine` ✅ | R11 | 12 | `state-store`, `run-context`, `recovery-engine`, `abort-controller`, `checkpoint-store` | `target-graph`, GRPH shape, durable execution for graphs |
| `pipeline-engine` ✅ | R11 | 12 | `ir-model`, `model-adapter`, `run-context` | `target-pipeline`, `query-router`, RAG composition |
| `durable-execution` ✅ | R11 | 12 | `checkpoint-store`, `run-context`, `idempotency-store` | GRPH/MGD/BATCH long runs, `research-checkpointer` |
| `eval-service` ✅ | R15 | 9 | `trace-event-bus`, `dataset-registry`, `grader-registry`, `runtime-orchestrator` | `eval-runner`, `eval-report`, `prompt-optimizer`, `target-eval-bundle` |
| `prompt-optimizer` ✅ | R15 | 9 | `eval-service`, `model-adapter`, `system-prompt-builder` | Active-eval optimization (Pillar 2) across all shapes |
| `gateway-server` ✅ | R16 | 17 | `secrets-manager`, `runtime-orchestrator`, `permission-engine` | `gateway-protocol`, `web-ui`, `studio-server`, MGD/CHN remote, `mcp-server-host` |
| `voice-runtime` ✅ | R16 | 17 | `model-adapter`, `audio-stream`, `vad-engine` | `target-voice`, `barge-in-controller`, `call-session`, `tool-tts-stt` |
| `tenancy` ✅ | R17 | 17 | `secrets-manager`, `audit-log`, `gateway-server` | MGD multi-customer isolation |
| `audit-log` ✅ | R17 | 17 | `event-log`, `secrets-manager` | `tenancy`, `policy-engine` audit story, MGD compliance |
| `computer-use-driver` ✅ | R18 | 20 | `infra-utils` | All R4 BROW tools, `target-browser-driver` |
| `research-planner` / `crawler` / `branch-explorer` / `report-synthesizer` / `research-checkpointer` ✅ | R19 | 20 | (per-row `Depends on`) | `target-research-bundle`, RES shape end-to-end |
| `batch-runner` ✅ | R20 | 20 | `queue-consumer`, `retry-policy`, `runtime-orchestrator` | `target-batch-worker`, `fan-out-fan-in`, `batch-eval-loop`, BATCH shape |

### Active v1.3 critical-path candidates

| # | Module | Layer | Candidate section | Why it matters | Direct downstream unblocked |
|---|---|---|---|---|---|
| 1 | 🔴 `plugin-sdk` v2 | F5 | §41 | Stable typed contract surface (semver-tracked) for third-party tools/channels/models/graders/target-backends. The §40 template marketplace gives users templates; §41 gives users *plugins*. Sequential prereq for §42. | `plugin-loader`, `module-marketplace-client`, third-party tool ecosystem |
| 2 | 🔴 `plugin-loader` | F5 | §41 | Runtime activation of third-party plugins with sandboxed import + capability gating. Reuses §31 plugin-sandbox content-isolation + §40 sigstore-style signature verification. | Production plugin loading, marketplace install path |
| 3 | 🟡 `module-marketplace-client` | F5 | §42 | Studio Marketplace tab for community plugins (mirroring §40 template-marketplace-client). | Community-published plugin distribution, ecosystem network effects |
| 4 | 🟡 `target-ios-bundle` | F2 | §43 | iOS Swift Package Manager bundle wrapping the Bun-on-iOS Hermes embedding. Substantial new compilation path. | Native iOS app embedding, consumer-mobile partners |
| 5 | 🟡 `target-android-bundle` | F2 | §43 | Android AAR bundle wrapping the Bun-on-Android NDK bridge. Substantial new compilation path. | Native Android app embedding, consumer-mobile partners |
| 6 | `cloud-adapter-{render,flyio,railway,heroku,vercel}` | F3 | §44 | Dev-friendly one-click deploy. Each ~200 LOC wrapping the platform's deploy API + a target-specific Dockerfile/runtime variant. | Fast onboarding for solo devs / OSS demos / weekend hacks |

### Sequencing observations (informational)

- `plugin-sdk` v2 and `plugin-loader` together unblock the entire third-party ecosystem. They're the highest-leverage v1.3 investment.
- The mobile target shapes (§43) are a deep, narrow chain: deferred until the underlying Bun runtime story matures.
- The cloud adapters (§44) are independent of each other and of the plugin SDK — they can ship opportunistically as partners or contributors materialise.

---

## Cross-cutting hardening notes

- **`tool-code-execution`** runs in `sandbox` with network=none / read-only-root defaults (since §18). `tool-bash` (R4) still operates at host trust level by design — it's the operator-controlled escape hatch. Any production EVAL or MGD deployment running untrusted samples should prefer `Python` / `JavaScript` / `Shell` (sandboxed) over `Bash` (host).
- **Boundary classifier** (`@crewhaus/boundary-classifier`) is the single chokepoint for untrusted content ingestion — see [Pillar 3 in AGENTS.md](https://github.com/crewhaus/factory/blob/main/AGENTS.md). Every site that pulls externally-controlled content into the model's context must classify before injecting.
- **Eval-driven mutations** must patch the *spec*, not the IR — see [Pillar 2 in AGENTS.md](https://github.com/crewhaus/factory/blob/main/AGENTS.md). `lower()` does destructive normalization that can't round-trip.

---

## Module count

- **Factory-level**: ~35 modules across F1–F5.
- **Runtime composable**: ~155 modules across R1–R20.
- **Total**: ~190 modules.
- **Grouped bundles**: 18 coarse aggregations (see [MODULE-CATALOG.md → Module groups](MODULE-CATALOG.md#module-groups)).
- **Shipped**: 180. **Remaining**: ~10 (all in v1.3+).

---

## Where to read next

- **Architectural shape of the catalog** — [MODULE-CATALOG.md](MODULE-CATALOG.md)
- **One-page brief per module** — [module-briefs/](module-briefs/README.md)
- **Section-by-section history with PR refs** — [build-roadmap.md](https://github.com/crewhaus/operations/blob/main/build-roadmap.md)
- **Task-oriented walkthroughs** — [walkthroughs/](https://github.com/crewhaus/demos/blob/main/walkthroughs/INDEX.md)
