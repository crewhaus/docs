# hangar-server

Status: implemented and tested
Dependency phase: 18 - Deployment & studio
Catalog layer: F3 - Deployment & Operations
Origin in ordering: 0.5.0 harness manager (Hangar M1–M4; factory PRs #372–#376)
Workspace home: packages/hangar-server
Targets: All (it operates compiled harnesses; it is not embedded in one)
Test layers: T1, T3, T8

## Purpose

The Hangar console's HTTP server: a token-gated loopback API over the machine-wide harness registry, and the thing `crewhaus hangar` boots. It answers fleet rows with cached rollups, per-harness detail (spec, preflight, sessions, evals, memory, cost), registry CRUD, supervised process control, a live SSE run feed, a `crewhaus.control.v1` proxy, the approvals and review inboxes, an activity digest, a bounded job queue, the M3 detail surface — 178 routes across eleven groups (`spec`, `memory`, `evals`, `data`, `feedback`, `creds`, `channels`, `security`, `thredz`, `inspect`, `runtime`) — and the Advisor + library-curation surface (group `advisor`, plus `PUT /api/h/:id/visibility` and `GET /api/registry/discover`).

The Advisor rides the same grouped dispatch table as the M3 routes (`src/advisor.ts`), so it inherits every guard uniformly, and three rules bound it. Nothing is invented: every feed item — severity `critical | warn | suggestion` — derives from a signal another panel already reads (preflight, the spec lint, eval health vs the pinned baseline, the cost fold vs the declared budget, incidents, parked approvals, overdue dreams, the advice feed); `deriveAdvisorItems` is pure, and the empty feed IS the goal state ("running optimally"). A suggestion is never an application: a quick action queues a CLI verb through the job queue with argv from the closed `ADVISOR_JOB_ARGV` vocabulary or deep-links the owning screen, and a submitted issue queues `optimize` / `eval` / `doctor` / `compile` / `advise` or records a `note` — the issue text never reaches a command line. A decision is a record, not a deletion: act (optional comment) and dismiss (REQUIRED reason) append to `<harness>/.crewhaus/advisor/decisions.jsonl`, reopen appends a superseding record, reports persist under `.crewhaus/advisor/reports/` (`REPORT_KINDS = model-usage | costs | usefulness | optimization`) and issues under `.crewhaus/advisor/issues.jsonl` — all reads capped and torn-line tolerant, and the feed GET derives without writing. Library curation is the registry's first-class `hidden` field surfaced on every `/api/harnesses` row (curation, not removal), and `GET /api/registry/discover` walks the scan roots reporting unregistered harnesses as candidates while registering nothing — distinct from `POST /api/scan`, which registers everything it finds.

The security posture is the design: bearer on every `/api` route, no cookies anywhere (hence no CSRF surface), constant-time token comparison over sha256 digests, and a single-use `/boot/<nonce>` handoff so the token reaches the browser as a URL fragment and never as a command-line argument.

## Boundaries

Owns:

- **`auth.ts`** — token minting into `<hangarRoot>/token` (0600, dir 0700, reused across boots), `tokenEquals` (sha256 + `timingSafeEqual`, so unequal lengths compare in the same time as equal ones), and the boot-ticket map: 256-bit nonces, `BOOT_TICKET_TTL_MS = 120_000`, consumed **before** expiry is checked so a ticket is spent even when stale. The redirect answers 302 to `/#t=<token>` with `cache-control: no-store` and `referrer-policy: no-referrer`.
- **`settings.ts`** — read-only mode. `GET`/`HEAD`/`OPTIONS` always pass; every other method on every other path is refused 403 with `{error, code:"read_only", method, path, locked, remedy}`. The exemption set is exactly two EXACT strings (`PUT /api/read-only`, `POST /api/boot-ticket`), not prefixes, because a prefix exemption widens silently every time a route is added underneath it. Enforcement sits after auth and before dispatch, so a new route is covered by construction. `locked` is load-bearing: the two postures have different remedies.
- **`process.ts`** — the supervised-process face over `@crewhaus/harness-supervisor` and the spawn-env stamps. The four HTTP job kinds (`doctor`, `compile`, `eval`, `dream-run`) and their closed argv vocabulary live in harness-supervisor's `./job-kinds`, and this file re-exports them. It stamps `CREWHAUS_CONTROL_PORT=0` and deliberately stamps **no** `CREWHAUS_CONTROL_TOKEN`, so the daemon mints its own and a dead daemon's token cannot authenticate against its replacement.
- **`control-client.ts`** — re-exports the control.v1 client for one release; the client moved to gateway-protocol's `./control-client`. It re-reads the 0600 token file on every call, parses the daemon's boot line for the real port, treats a literal `0` as "not known yet", and classifies every refusal into the eight-code table with its `retryable` / `expected` flags.
- **The per-area modules** — `runtime-ops`, `creds-ops`, `memory-ops`, `evals-ops`, `security-ops`, `wiki-ops`, `spec-edit`, `builders`, `thredz`, `inspect`, `activity`, `notifications`, `omnibox`, `onboarding`, `rollups`, `runs`, `deployments`, `plugins` — each owning one route group, each read answering `{present, note, verb}` — and `advisor`, whose group rides the same dispatch table and whose empty feed is the first-class "running optimally" state rather than an empty envelope. `creds-ops` keeps thin writer wrappers and `runs` keeps a `runDetail` wrapper, because the masker and `HttpError` stay at the server's exit; `env-file` and `safety` re-export.
- **The caps.** Every JSONL reader is bounded and torn-line tolerant (`MAX_JSONL_LINES = 5000`, `MAX_JSONL_BYTES = 8 MiB`, `MAX_RAW_LINES = 1000`, `MAX_TEXT_BYTES = 256 KiB`, `MAX_MEMORY_ITEMS = 500`, …), and truncation is reported rather than hidden. `SSE_IDLE_TIMEOUT_SECONDS = 255` is Bun's maximum, chosen because Bun's 10 s default severs exactly the quiet `heartbeat: every 60s` streams the console exists to hold open.

Does not own: the CLI verb surface (`apps/cli/src/hangar-cmd.ts` owns the lock file, the flag parsing, the remote-bind gate and the boxed summary); the registry file format (`harness-registry`); process supervision itself (`harness-supervisor`); the checks (`preflight`); the browser assets (`hangar-ui`); and any write path that another package already owns — spec edits go through `applySpecEdits` with `restrictToOptimizable`, env through `upsertEnvVar` (harness-supervisor's), memory through tombstones and trash directories, never a hard delete.

## The assistant

Status: planned

An optional panel in Home, the 0.9.0 name for Hangar's web console: a person opens it from any object and asks about what they are viewing. It is off until an owner turns it on, and it answers from that harness's own records, read as the asking member. It is a module of this package with no spec key, IR field, emitter or harness change, because its only consumer is this server and it never enters a bundle.

- **The module.** `src/assistant/`, handlers in `v1/assistant.ts`, the release switch in `src/assistant/release.ts` and the deterministic suite in `src/assistant/redteam.test.ts`, which runs on every PR against a scripted model that obeys any instruction it sees. A live red-team run sets the switch before the release tag.
- **Context.** Each stored record is classified at TrustOrigin `record` before it can enter the model's context, and the member's question at `api`. A malicious piece never reaches the model. Content marked `people` and model thinking are never included, for any role, and are listed as withheld.
- **Read tools.** A closed list of the API's read operations, each through the in-process `invoke`, which refuses any operation that writes, authorizes as the asking member and applies the exit masker. None writes, starts a process or reaches the network.
- **Suggestions.** A `suggest` tool records a proposal for the member, who acts with the ordinary button, permission and receipt. A proposal whose permission is on the never-propose list is dropped and counted.
- **Records.** The server holds each conversation in memory, so no client can forge an earlier turn, and nothing about it reaches disk but a content-free usage line. Daily caps bound each member, and cost reads "about" a figure or "cost unknown", never `$0`.
- **Permission.** `home.assistant.use`, held by operators and above, for members only.

New dependencies: `@crewhaus/boundary-classifier` and `@crewhaus/run-context`. It lands with `feat(hangar-server): a read-only assistant beside every object, behind its redteam gate` ([#523](https://github.com/crewhaus/factory/issues/523)).

## The assistant's confirmed actions

Status: planned

Off until an owner turns on an assistant rule, one of the delegation rules in brief 325. The rule bounds what the assistant may carry out, never what the member may do.

- **`act`.** With an active rule covering the conversation's harnesses, the model gains this tool, which proposes a plan of at most five operations from the closed `ASSISTANT_ELIGIBLE` list. It is withheld in a turn whose context holds any piece classified suspicious, and the panel says so. Nothing needing `h.decision.resolve` is eligible, so the assistant never touches a decision.
- **A server-held plan.** The server checks each operation and keeps the plan with the conversation under `planDigest`. The card comes from the checked operations and fixed templates; the model's words appear only as its labelled answer around it.
- **Confirmation.** **Do it** sends the digest, and the server checks everything again, then runs each operation as the member through the router's receipted dispatch. `invoke` stays read-only and is never used to write. A replayed, edited or foreign digest is refused.
- **Receipts.** Each operation writes its own receipt, and `assistant.act` writes one more naming the conversation, the digest and those receipts. Each names the member as its actor and carries `authority {kind: "assistant-rule", rule, revision}`. `via` keeps [#507](https://github.com/crewhaus/factory/issues/507)'s list and says where the member confirmed.
- **No held-out gate or rollback,** unlike the rules that merge, roll out or adopt: the member decides each plan with their own authority, and each eligible operation can be undone or stays narrow.
- **Its own release switch,** `ASSISTANT_ACT_RELEASED`, beside the assistant's. With only this one off, `act` is gone and the read-only assistant is unchanged.

It lands with `feat(hangar-server): the assistant can carry out a confirmed action within your authority and an owner's rule` ([#523](https://github.com/crewhaus/factory/issues/523)).

## Inputs and Outputs

Inputs: a registry handle, a hangar root, an optional static-asset map, a bind host/port, the auth posture (token or `noAuth`), the read-only posture, and an injected env.

Outputs: a `HangarServer` (`url`, `token`, `tokenPath`, `registryPath`, `processes`, `stop()`), JSON over 178+ routes, and `text/event-stream` run feeds whose grammar is `replay` first and `done` last, so a finished run and a live one take the same client code path.

## Dependency Notes

Depends on `@crewhaus/harness-registry`, `@crewhaus/harness-inventory`, `@crewhaus/harness-supervisor`, `@crewhaus/preflight`, `@crewhaus/gateway-protocol`, and the stores it reads through (`session-store`, `event-log`, `memory-store`, `continuity-store`, `wiki-store`, `dataset-registry`, `eval-report`, `feedback-distill`, `audit-log`, `data-retention-engine`, `plugin-loader`/`plugin-sdk`). It does **not** depend on `@crewhaus/secrets-manager`, which is why `POST /api/h/:id/secrets/:name/rotate` is the one route that answers 501 — and why it refuses the CLI fallback, which would put the value in argv.

`server.stop()` deliberately leaves children alone, and an attached job child keeps the host event loop alive, so `stop()` is not by itself an exit — `crewhaus hangar` calls `process.exit` explicitly after the shutdown report.

The manager never `import()`s plugin code: it holds every harness's `.env` chain. `traceObservers()` therefore publishes the eligibility set (plugin names) rather than delivering events, and pane documents are served capped at 256 KiB into an iframe with `sandbox="allow-scripts"` and no `allow-same-origin`, under a fail-closed CSP derived from the plugin's own `net` allow-list.

## First Implementation Slice

Shipped across the four Hangar milestones: M1 the read-only console and `crewhaus hangar` (#372), M2 the driver surface and `crewhaus daemon` (#373), M3 the 178-route detail surface (#374), M4 shutdown, the §10 decisions and polish (#375), plus the gating Windows supervision CI job (#376).

## Study References

`studio-server` (the authoring-side precedent this deliberately does not extend), the run-console contract in `@crewhaus/ui` (the same balanced-JSON `TraceEvent` splitter), and the read-only-mode discussion in `settings.ts`, which states plainly that the bearer token — not the toggle — is the security boundary.

## Validation Plan

Catalog tests: T1, T3, T8. Primary risks: **a read that leaks** — an unmasked credential in a memory fact or a symlink escaping the harness directory (both closed: every payload passes the credential masker, every file read is realpath-contained per file, and the raw `logs/<runId>.log` is excluded from the inspector entirely) — and **route-map drift** between the server's table and the browser's generated wrappers.

Definition of done: the contract test asserts `M3_ROUTES` and `routes.js` are the same set key-for-key, method-for-method, path-for-path, then drives every route against a live fixture server (a 404 there means dispatch drift, a 500 means a guard threw); every M3 GET carries a `{present, note, verb}` table; and `crewhaus hangar serve --smoke` passes against the compiled binary.
