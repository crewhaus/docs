# Module Catalog — implementation status

This document tracks **what's built, what's in flight, and what's deferred** across the catalog. It complements [MODULE-CATALOG.md](MODULE-CATALOG.md), which describes the catalog's shape and is intended for navigation, not for ticket-tracking.

If you want the *what / why / how* of a module, go to [MODULE-CATALOG.md](MODULE-CATALOG.md) or [module-briefs/](module-briefs/README.md). If you want section-by-section history with PR refs, go to [build-roadmap.md](build-roadmap.md). For shipped + planned product surface in user-facing terms, see [factory ROADMAP.md](https://github.com/crewhaus/factory/blob/main/ROADMAP.md). This document sits between them: a single-glance status board.

---

## Snapshot — 2026-05-26 (post-§41/§42/§44 close-out + §55–§59 integration batch)

- **v0.1.0 launched 2026-05-15** — public release covering build-roadmap §1–§40 (compiler core, twelve target shapes, eval stack, security fabric).
- **v0.2.x in flight (Days 30–60)** — reference-corpus integrations: §51 egress fabric, §52 context curation, §53 justification gates + 12-metric rubric, §54 codegraph tool. Most v0.2.x packages have landed in `factory/packages/`; the slate is closing out polish + recipes.
- **§41–§42 plugin extension surface shipped 2026-05-26** — plugin-sdk v2, plugin-loader, plugin-registry, module-marketplace-client. Closes out the highest-leverage v1.3 investment per the prior status doc.
- **§44 cloud-deploy adapters shipped 2026-05-26** — cloud-adapter-{render, flyio, railway, heroku}.
- **§55–§59 integration batch shipped 2026-05-26** — eight tracks (A–H) plus cross-cutting work derived from external research (5 arxiv papers + 4 blog studies + 3 reference repos). Six new packages and three pull-through changes; see new §55–§59 rows in [Implementation summary](#implementation-summary).
- **~152 of ~205 catalog rows implemented**, including the six new §55–§59 packages.
- **Target shapes ship today** — cli / workflow / channel / graph / managed / pipeline / crew / research / batch / voice / browser / eval / onchain / onchain-game. The §59 `target-claude-plugin` emitter transforms any of these into an Anthropic-compatible Claude Code plugin directory.
- `bun run tsc -b` clean. `biome check .` clean. `bun run test:smoke` (compile-time matrix across target shapes) green. 5896 unit tests pass.

The remaining ~53 catalog rows are split between v0.2.x cleanup, the §43 mobile target shapes (deferred), and a long tail of low-leverage runtime modules. See [Unbuilt module inventory](#unbuilt-module-inventory) below.

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

## Versioning crosswalk

| Project version | Build-roadmap sections | Status |
|---|---|---|
| **v0.1.0** | §1–§40 | Shipped 2026-05-15. Compiler core, 12 target shapes, eval stack, security fabric (boundary classifier), production hardening, distribution. |
| **v0.2.x** | §51–§54 + §46 (active IR-patch optimizer) + §47 (onchain slices 0–2) | In flight (Days 30–60). Reference-corpus integrations: egress fabric, context curation, justification gates, 12-metric rubric, codegraph tool. Most packages landed; recipes + polish remaining. |
| **v0.3.x** | (TBD — see factory ROADMAP) | Crewhaus Forge community registry, `crewhaus publish`, verified-artifact distinction. |
| **v0.4.x+** | §41–§45 + new sections | Plugin SDK v2, plugin loader/registry, module marketplace, cloud-deploy adapters, additional channel adapters, optional non-EVM chain adapters. |
| **v1.0** (target: Q4 2026) | Stabilization | Schema + CLI commands stable, security audit complete, production-readiness statement per target shape. |

Per the [factory ROADMAP](https://github.com/crewhaus/factory/blob/main/ROADMAP.md), pre-1.0 minors may include breaking changes documented in the changelog.

---

## Implementation summary

Sections 1–40 (v0.1.0) are closed. Sections 46, 47, 51–54 (v0.2.x) have landed package-side; recipes + polish are the remaining v0.2.x slice. The build-roadmap is the section-by-section authority; this list groups what each section delivered so a contributor can locate a module's history quickly.

| Sections | Phase | Delivered |
|---|---|---|
| §1–§5 | v0.1.0 — CLI core | spec / compiler / IR / runtime-core / tool framework / target-cli |
| §6 | v0.1.0 — workflow target | `target-workflow`, `IrWorkflowV0`, per-step model overrides |
| §7–§11 | v0.1.0 — safety + extension surface | permission-engine, hooks-engine, skills-registry, slash-commands |
| §12–§14 | v0.1.0 — channel + multi-modal | target-channel-bot, Slack adapter, web/image/fetch tools |
| §15 | v0.1.0 — observability | trace-event-bus, otel-exporter, metrics-collector, structured-event-printer |
| §16 | v0.1.0 — eval stack | eval-dataset, eval-grader, eval-judge, eval-runner, eval-report, `crewhaus eval` |
| §17 | v0.1.0 — multi-provider models | adapter-openai/gemini/bedrock, model-router with lazy SDK loading |
| §18 | v0.1.0 — safety floor | sandbox, tool-code-execution, prompt-injection-detector |
| §19 | v0.1.0 — graph target | target-graph, graph-engine, checkpoint-store, branch-history, durable-execution |
| §20 | v0.1.0 — managed target | target-managed, gateway-protocol/server, policy-engine, tenancy, audit-log |
| §21 | v0.1.0 — RAG target | target-pipeline, pipeline-engine, chunker, embedder, vector-store, tool-retrieve |
| §22 | v0.1.0 — crew target | target-crew, crew-orchestrator, agent-handoff, a2a-protocol |
| §23 | v0.1.0 — research + batch | target-research-bundle, target-batch-worker, queue-protocol, citation-tracker |
| §24 | v0.1.0 — voice target | target-voice, voice-runtime, vad-engine, barge-in-controller, call-session |
| §25 | v0.1.0 — browser target | target-browser-driver, computer-use-driver, tool-screen-capture/-mouse-keyboard/-vision-grounding |
| §26 | v0.1.0 — studio | wizard, scaffold-templates, plugin-sdk v0, studio-server, studio-ui, trace-viewer, graph-visualizer (sibling `crewhaus/utilities` repo) |
| §27 | v0.1.0 — production hardening | cost-tracker, secrets-manager, prompt-cache-manager, rate-limiter |
| §28 | v0.1.0 — deployment | spec-registry, ir-passes, migration-engine/runner, deployment-controller, canary-controller |
| §29 | v0.1.0 — eval depth | dataset-registry, grader-registry, regression-runner, prompt-optimizer, target-eval-bundle |
| §30 | v0.1.0 — backend adapters | SQS / Redis / Postgres queue adapters; Lance / Qdrant / Pinecone / Weaviate vector backends; Twilio / LiveKit telephony; Vapi realtime |
| §31 | v0.1.0 — studio v1 | Lit + Monaco + live SSE replay + plugin sandbox content isolation |
| §32 | v0.1.0 — distribution | docker-images, single-binary-cli, helm-chart, crewhaus-cloud (terraform + kustomize) |
| §33 | v0.1.0 — channel breadth | telegram / discord / whatsapp / imessage adapters |
| §34 | v0.1.0 — federation | federation-protocol / -discovery / -router (mTLS + sigstore-style pinning) |
| §35 | v0.1.0 — IDE & DX | vscode-extension, jetbrains-plugin, crewhaus-playground (sibling utilities repo) |
| §36 | v0.1.0 — polyglot sandboxes | go / rust / java / ruby / r / dotnet / php sandbox images on a registry pattern; `sandbox-image-registry` |
| §37 | v0.1.0 — vendor telemetry | datadog / honeycomb / splunk / newrelic exporters with credential-leak guards |
| §38 | v0.1.0 — production graders | NLG metrics, semantic similarity, safety classifiers, multimodal graders |
| §39 | v0.1.0 — compliance hardening | pii-redactor, audit-encryption, data-retention-engine, compliance-controls + `crewhaus compliance evidence` CLI |
| §40 | v0.1.0 — template marketplace | template-registry, template-marketplace-client, example-corpus CI matrix gate |
| §46 | v0.2.x — active IR-patch optimizer | spec-patch, prompt-optimizer-claude, eval-optimizer-orchestrator, `crewhaus optimize` |
| §47 | v0.2.x — onchain slices 0–2 | chain-adapter-base/-evm, tool-evm, tool-evm-tx, tool-contract-gateway, wallet-engine, permission-tokengated, target-onchain, target-onchain-game |
| §51 | v0.2.x — egress fabric (Pillar 3) | egress-classifier, tool descriptor `scope`, `RunContext.dataLineage`, `egress_decision` audit kind |
| §52 | v0.2.x — active context curation (Pillar 2) | compaction-curator, `OPTIMIZABLE_PATHS` extensions for `compaction.curate*` |
| §53 | v0.2.x — canonical eval rubric (Pillar 2) | grader-12-metric-rubric, `summarize12MetricRubric` cross-sample roll-up, `costPerUsefulOutput` aggregator |
| §54 | v0.2.x — AST-aware code intelligence | tool-codegraph (`CodeGraphSearch/Callers/Callees/Impact`) wrapping `@colbymchenry/codegraph` |
| §41 | 2026-05-26 — plugin SDK v2 + loader | plugin-sdk (typed surface for tools / channels / models / graders / target emitters + Ed25519 manifest signatures), plugin-loader (path allow-list + signature verification + capability gating) |
| §42 | 2026-05-26 — plugin discovery + marketplace | plugin-registry (file-backed JSON with secrets-resolvable trust anchors), module-marketplace-client (search / install / update / draftPublish over an abstract `ModuleRegistrySource`) |
| §44 | 2026-05-26 — cloud-deploy adapters | cloud-adapter-{render, flyio, railway, heroku} — config emission + Dockerfile + Deploy-API client per platform, all with the T8 credential-leak guard from §37 |
| §55 (Tracks A+B+C+D) | 2026-05-26 — failure taxonomy + arbiter + verifier synthesis + 12-metric coverage | `spec.failure_taxonomy` + `IrFailureTaxonomy` + recovery-engine `matchNamedFailure`; `eval-optimizer-orchestrator/failure-arbiter` (bug/spec-gap/noise/contract-ambiguity); `tool-harness-synthesizer` (Thompson-sampled tree search per AutoHarness 2603.03329); 12-metric coverage test guard. Sources: NLAH 2603.25723, Meta-Engineering Harnesses 2605.25665, AutoHarness 2603.03329, TDS 12-Metric. |
| §56 (Track E) | 2026-05-26 — meta-harness optimizer (**BREAKING**, opt-in) | `meta-harness-optimizer` — filesystem-backed full-history coding-agent proposer. Default mutator stays `rule`; this is `--mutator meta-harness`. Bundle output diverges from spec; run-header comment marks the divergence. Source: Meta-Harness 2603.28052. |
| §57 (Track F) | 2026-05-26 — typed graph DSL + runtime feedback channels | `IrMessageSchema` + `IrSchemaRef` on `IrCrewV0`/`IrGraphV0`; `wellFormednessCheck` pass in `ir-passes`; four new TraceEventBus event kinds (test_verdict, program_output, coverage_report, sanitizer_report). Source: AgentFlow 2604.20801. |
| §58 (Track G) | 2026-05-26 — contract compilation + specializations | `contract-compiler` (two-pass: completeness + ambiguity); `specialization-registry` (payments, auth, booking built-ins; project-local JSON overrides under `.crewhaus/specializations/`). Source: Meta-Engineering Harnesses 2605.25665. |
| §59 (Track H) | 2026-05-26 — `target-claude-plugin` emitter | `target-claude-plugin` package emits an Anthropic-compatible plugin directory (plugin.json + skills/ + agents/ + optional .mcp.json) from any IR variant. Source: claude-plugins-official (Anthropic reference repo). |
| §59-cross (Track 10) | 2026-05-26 — cross-cutting integrations | `RunContext.agentIdentity` (per-skill / sub-agent / role identity for audit); `rules-engine` (multi-language rule packs from `rules/{common,typescript,...}` with `CREWHAUS_RULES_PROFILE=core\|standard\|full` gating). Sources: Identity Governance; ECC §4.1. |

For per-section detail (file paths, kickoff prompts, PR refs), see [build-roadmap.md](build-roadmap.md). For user-facing release notes see [factory CHANGELOG.md](https://github.com/crewhaus/factory/blob/main/CHANGELOG.md).

---

## Unbuilt module inventory

This section replaces the prior `v1.3 backlog` with a complete inventory of every catalog row that does not yet have a working `factory/packages/<name>` entry, organized by layer. None of these block a shipped target shape; the project is feature-complete for v0.1.0 + v0.2.x. The order below is roughly by leverage — higher entries unblock more downstream work or more target shapes.

### F-layer (factory-level)

**F2 — Compiler & Codegen**
- 🟡 `bundle-packager` — Docker/OCI/npm/pypi packaging.

**F3 — Deployment & Operations**
- 🟡 `deployment-profiles` — local-dev / k8s / lambda / agent-engine / foundry / agentcore / hybrid-vpc.
- 🟡 `upgrade-controller` — in-place runtime upgrades.
- ~~`cloud-adapter-{render,flyio,railway,heroku}` — §44~~ **shipped 2026-05-26** via factory PRs [#118](https://github.com/crewhaus/factory/pull/118), [#119](https://github.com/crewhaus/factory/pull/119), [#120](https://github.com/crewhaus/factory/pull/120).

**F5 — Plugin SDK & Extension** *(all shipped 2026-05-26 — see [Implementation summary](#implementation-summary) §41 and §42 rows)*
- ~~`plugin-sdk` v2 — §41~~ shipped via factory PR [#121](https://github.com/crewhaus/factory/pull/121).
- ~~`plugin-loader` — §41~~ shipped via factory PR [#122](https://github.com/crewhaus/factory/pull/122).
- ~~`plugin-registry` — §42~~ shipped via factory PR [#123](https://github.com/crewhaus/factory/pull/123).
- ~~`module-marketplace-client` — §42~~ shipped via factory PR [#124](https://github.com/crewhaus/factory/pull/124).

**F2 — Mobile targets (deferred indefinitely)**
- `target-ios-bundle` — Swift Package Manager bundle around Bun-on-iOS Hermes embedding.
- `target-android-bundle` — Android AAR bundle around the Bun NDK bridge.

Deferral conditions: (1) Bun publishes a stable iOS embedding API, (2) Android NDK bridge ships in Bun mainline, (3) at least one external partner signs an LOI, (4) CI runners with `xcodebuild` + `gradle` are provisioned.

### R-layer (composable runtime)

**R1 — Runtime Core** — 🟡 `query-engine`, 🟡 `stream-runtime`, 🟡 `scheduler`, 🔴 `durability-mode`

**R2 — Model Layer** — 🟡 `response-format-coercion`, 🟡 `reasoning-controller`, 🟡 `speculation-engine`, 🟡 `auth-profiles`

**R3 — Tool Layer (core)** — 🟡 `tool-search`, 🟡 `tool-display`

**R4 — Built-in Tools** — 🟡 `tool-process`, 🔴 `tool-browser`, 🔴 `tool-tts-stt`, 🟡 `tool-ask-user`, 🟡 `tool-cron`, `tool-plan-mode`, `tool-worktree`, 🟡 `tool-lsp`, 🟡 `tool-monitor`, `tool-sleep`, `tool-notebook-edit`, 🟡 `tool-notification`, 🟡 `tool-remote-trigger`, 🟡 `tool-canvas`, 🟡 `tool-citations`, 🟡 `tool-dom-inspector`

**R5 — MCP & Protocol Hosts** — 🟡 `mcp-server-host`, 🟡 `acp-protocol`, 🟡 `ag-ui-protocol`, 🟡 `webhook-host`, 🟡 `chain-adapter-solana`, 🟡 `chain-adapter-cosmos`, 🟡 `chain-adapter-bitcoin`

**R6 — Context & Memory** — 🟡 `context-engine`, 🟡 `compaction-microcompact`, 🟡 `compaction-context-collapse`, 🟡 `compaction-reactive`, 🟡 `compaction-tool-result-budget`, 🟡 `compaction-session-memory`, 🟡 `bootstrap-files`, 🟡 `system-prompt-builder`, 🔴 `memory-service`, 🟡 `memory-extraction`, 🟡 `personalization-store`

**R7 — State / Sessions** — 🟡 `app-state-store`, `global-state-singleton`, 🟡 `artifact-store`, 🟡 `session-router`, 🟡 `session-binding`, 🟡 `replay-store`

**R8 — Permission / Policy / Safety** — 🟡 `approval-classifier`, `denial-tracking`, 🟡 `guardrails`, `safety-prompt-injector`, 🟡 `hitl-engine`

**R9 — Hooks / Skills / Slash** — `hook-loader`, `skills-serializer`, 🟡 `output-styles`, 🟡 `autoreply-policy`

**R10 — Multi-Agent** — 🟡 `subagent-runtime`, 🟡 `coordinator`, 🟡 `swarm-runtime`, 🟡 `handoff-engine`, 🟡 `role-system`, 🟡 `task-engine`, 🟡 `pairing-engine`

**R11 — Workflow / Graph / Pipeline** — 🔴 `workflow-engine`, 🟡 `workflow-checkpointer`, 🟡 `event-bus`, 🟡 `checkpoint-encoder`

**R12 — RAG** — 🟡 `document-store`, 🟡 `retriever`, 🟡 `ranker`, 🟡 `reader-extractor`, 🟡 `query-router`, 🟡 `generator`, `document-loader`, 🟡 `knowledge-graph`, 🟡 `ingestion-pipeline`

**R13 — Channels** — `channel-registry`, 🟡 `channel-{signal,bluebubbles,email,sms,web}`, `channel-transport`, `channel-binding`, 🟡 `channel-router`, `channel-features`, `directory-adapters`, 🟡 `media-payload`, 🟡 `pairing-flow`, `message-actions`

**R14 — Scheduling** — 🟡 `scheduler-cron`, 🟡 `heartbeat-engine`, 🟡 `isolated-agent-runner`, `session-reaper`, `background-housekeeping`, 🟡 `task-scheduler`, `retry-policy`, 🟡 `dead-letter-queue`, `batch-progress-tracker`

**R15 — Telemetry / Eval** — 🟡 `telemetry-engine`, 🟡 `trace-recorder`, 🟡 `replay-engine`, 🟡 `benchmark-runner`, 🟡 `trajectory-grading`, 🟡 `canary-router`, 🟡 `cost-attribution`

**R16 — UI / Voice / Media** — 🟡 `tui-runtime`, `tui-keybindings`, 🟡 `repl-launcher`, 🟡 `web-ui`, 🔴 `audio-stream`, 🟡 `dtmf-handler`, 🟡 `prosody-controller`, 🟡 `media-service`, 🟡 `canvas-host`, 🟡 `notifications-service`

**R17 — Infrastructure** — `config-loader`, 🟡 `settings-sync`, `i18n`, `feature-flags`, 🟡 `runtime-migrations`, 🟡 `daemon-process`, 🟡 `node-host`, 🟡 `proxy-capture`, 🟡 `bootstrap-runtime`, `startup-profiler`, 🟡 `update-channel`

**R18 — Specialized** — 🔴 `browser-extension-bridge`, 🟡 `lsp-host`, 🟡 `sandbox-fs-overlay`, 🟡 `structured-output-tools`, 🟡 `autoplan-engine`, 🟡 `auto-classifier`, `fingerprint`, `activity-manager`, `speculation-cache`, 🟡 `commitments-engine`

**R19 — Research** — 🔴 `branch-explorer`, 🟡 `evidence-store`, 🟡 `progress-streamer`, 🟡 `source-quality-scorer`

**R20 — Batch** — 🟡 `batch-job-spec`, 🟡 `fan-out-fan-in`, 🟡 `output-sink`, 🟡 `batch-eval-loop`

**Total unbuilt rows**: ~115. **None are critical-path for any shipped target shape.**

---

## Active critical-path candidates

Cross-references the per-layer `Depends on` columns in the catalog with the build phases. This section names every unbuilt module whose absence would stall a target shape, a stack, or another high-leverage module.

> **Closed out 2026-05-26**: the §41 plugin SDK + loader, §42 plugin registry + marketplace client, and §44 cloud-deploy adapters all shipped via factory PRs #118–#124. They are removed from this table — see the Implementation summary above for what landed.

| # | Module | Layer | Candidate section | Why it matters | Direct downstream unblocked |
|---|---|---|---|---|---|
| 1 | 🔴 `workflow-engine` | R11 | future | Async-event workflow runtime; pub/sub event bus, step decorators. | `workflow-checkpointer`, `event-bus`, `checkpoint-encoder`, MGD long-runs |
| 2 | 🔴 `memory-service` | R6 | future | Long-term memory across CHN / CRW / GRPH / RES. | `memory-extraction`, `tool-memory` deeper integration |
| 3 | 🔴 `durability-mode` | R1 | future | Configurable durability: `exit` / `async` / `sync` checkpoint write. | GRPH/MGD/CHN production durability tiers |
| 4 | 🔴 `audio-stream` | R16 | future | PCM/Opus I/O streaming; buffering. | `tool-tts-stt` end-to-end, voice production hardening |
| 5 | 🔴 `branch-explorer` | R19 | future | Multi-branch research with prune. | RES production-grade depth |
| 6 | 🔴 `tool-browser` | R4 | future | Stateful Playwright/Chromium automation; pairs with `target-browser-driver` for non-BROW shapes. | CLI / CHN / CRW / RES browser actions |
| 7 | 🔴 `browser-extension-bridge` | R18 | future | Bridge between harness and a browser extension MCP. | Hybrid BROW deployments where the daemon doesn't own the page |

### Sequencing observations

- ~~`plugin-sdk` v2 and `plugin-loader` together unblock the entire third-party ecosystem.~~ **Closed out 2026-05-26.** The third-party ecosystem is now unblocked from the SDK side; the next investment is community-published plugins.
- Mobile targets (§43) are deferred indefinitely — see [Unbuilt module inventory](#unbuilt-module-inventory).
- ~~Cloud adapters (§44)~~ **closed out 2026-05-26.** All four (Render, Fly.io, Railway, Heroku) shipped; a future cloud-adapter-vercel still waits on a `target-vercel-functions` shape not yet in the catalog.
- `workflow-engine` + downstream is now the largest single buildout in the post-§41/§42/§44 backlog; it adds an alternative to `graph-engine` for async-event topologies (vs. the synchronous turn-based graph today).

---

## Cross-cutting hardening notes

- **`tool-code-execution`** runs in `sandbox` with network=none / read-only-root defaults (since §18). `tool-bash` (R4) still operates at host trust level by design — it's the operator-controlled escape hatch. Any production EVAL or MGD deployment running untrusted samples should prefer `Python` / `JavaScript` / `Shell` (sandboxed) over `Bash` (host).
- **Boundary classifier** (`@crewhaus/boundary-classifier`) is the single chokepoint for untrusted content *ingress* — see [Pillar 3 in AGENTS.md](https://github.com/crewhaus/factory/blob/main/AGENTS.md). The v0.2.x **egress-classifier** is its sink-side counterpart. Every site that pulls externally-controlled content into the model's context must classify before injecting; every external sink must classify against `RunContext.dataLineage` before emitting.
- **Eval-driven mutations** must patch the *spec*, not the IR — see [Pillar 2 in AGENTS.md](https://github.com/crewhaus/factory/blob/main/AGENTS.md). `lower()` does destructive normalization that can't round-trip. The v0.2.x `spec-patch` package + `eval-optimizer-orchestrator` wire the full active loop.

---

## Module count

- **Factory-level**: ~37 catalog rows across F1–F5.
- **Runtime composable**: ~170 catalog rows across R1–R20 (including v0.2.x additions).
- **Total**: ~205 catalog rows.
- **Grouped bundles**: 18 coarse aggregations (see [MODULE-CATALOG.md → Module groups](MODULE-CATALOG.md#module-groups)).
- **Shipped (✅)**: ~152. **Unbuilt**: ~53 (excluding the deferred mobile targets).
- The `factory/packages/` directory contains 184 workspace packages — slightly higher than the ✅ count because some catalog rows aggregate multiple packages (e.g., R4 `sandbox-image-{python,javascript,...}` is one row, multiple packages) and a few packages are scaffolding/testing infrastructure rather than catalog modules (e.g., `smoke-harness`).

---

## Where to read next

- **Architectural shape of the catalog** — [MODULE-CATALOG.md](MODULE-CATALOG.md)
- **One-page brief per module** — [module-briefs/](module-briefs/README.md)
- **Section-by-section history with PR refs** — [build-roadmap.md](build-roadmap.md)
- **User-facing release notes** — [factory CHANGELOG.md](https://github.com/crewhaus/factory/blob/main/CHANGELOG.md) and [ROADMAP.md](https://github.com/crewhaus/factory/blob/main/ROADMAP.md)
- **Task-oriented walkthroughs** — [walkthroughs/](https://github.com/crewhaus/demos/blob/main/walkthroughs/INDEX.md)
