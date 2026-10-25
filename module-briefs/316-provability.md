# provability

Status: implemented and tested
Dependency phase: 9 - Telemetry & eval
Catalog layer: R15 - Telemetry, Tracing, Eval
Origin in ordering: 0.8.0 — the provability release (design §2, §5, §6)
Workspace home: packages/provability
Targets: CLI, CHN, MGD, RES (the shapes whose `provability:` block is wired; crew carries the block with an `accepted-but-unwired` warning, and every other shape rejects the key)
Test layers: T1, T8

## Purpose

The deterministic half of the provability ladder. A spec sets a floor — `cited`, `grounded`, `checked` or `proven` — and every claim in an answer must bind to evidence at or above it. This package is the evidence index built from the run's own event log, the four verifiers behind one function (`verifyDocument`), the proof bundle they produce, the ways to render it, and where bundles and proof artifacts are stored.

It makes no model call and imports no model adapter, and `crewhaus doctor --philosophy-alignment` checks that. The three things that reach outside the process — the HEAD request behind `sources.resolve: head`, the replay of a pure tool, and the opt-in paraphrase-waiver judge — are callbacks the caller injects. The runtime gate, the `ProvabilityPreflight` tool, `crewhaus provability check` and the research report gate all call the same `verifyDocument`, so a draft the model preflights is graded exactly as the gate will grade it.

## Boundaries

Owns:

- **The four rungs, as checked** (`verify.ts`):
  - **`cited`** — every marker in a claim has a definition, and the definition names something locatable: an `http(s)` URL or a `doi:` whose host passes `sources.allow` / `sources.deny` and is not a private address, a workspace path that stays inside the workspace and exists, or, only under `sources.bibliographic: true`, free text. With `sources.resolve: head` each URL must also answer one HEAD with 2xx or 3xx.
  - **`grounded`** — the cited origin was read in this run (a non-error result, answered inside 2xx), and claim-ledger's `locateClaim` finds the claim in that body at `grounded.coverage` within `grounded.window_tokens`. A figure or a negation is never excused by a partial bar, a quotation must appear verbatim, and `grounded.independent: n` asks for n distinct origins.
  - **`checked`** — `tool:<toolUseId>` resolves to a recorded, non-error call of a tool on `execution.allow`, the graded session's own call first, then a sub-agent's. Every number the claim states must be in the result as an exact decimal and every quotation word for word; a claim with neither must share `grounded.coverage` of its words with the call. What the call's input handed over is blanked out of the result first, so a tool that only hands the claim back checks nothing, and a `FactCrossCheck` result counts only where it found this sentence `supported`. When the tool is pure and `execution.replay` is on, it is re-run and the digests compared.
  - **`proven`** — `proof:prf_…` names a stored artifact that a checker on `formal.checkers` accepted. Because an artifact is a workspace file the model's own tools could write, the run's evidence must also hold the `ProofCheck` call that wrote it, given exactly the checker, claim, statement and proof the file holds, and where a re-check is wired the builtin checker accepts the stored proof again. The sentence must state the artifact's recorded claim: its words at `grounded.coverage`, the same figures, and each word denied in both or in neither. The checker, the formal statement, the hypotheses the proof rests on and its axioms go into the bundle beside the prose — the formalization gap is shown, not closed.

  Each shortfall is one stable `UnmetReason` token (`no_definition`, `not_fetched`, `value_absent`, `replay_mismatch`, `claim_mismatch`, …), which is what the retry nudge, the graders and `provability audit` read.
- **Assumptions** — `[A1]: assume: …` lines and a proof's hypotheses are counted on every surface and left out of the score on both sides (`score = met / (claims − assumptions)`). `assumptions: forbid` and `max_assumptions` refuse them, and a refusal fails the verdict whatever the score.
- **The evidence index** (`evidence.ts`) — `evidenceFromSessionLog` reads the session log and its sub-agent children (through their `sub_agent_start` brackets) in call order, `evidenceFromMessages` reads the turn still in flight, and `buildEvidenceIndex` merges the same call from both. Evidence comes from the log rather than the conversation, so a fetch compaction evicted from the messages still grounds. Origins are read off a call only where its result is that origin's content — `WebFetch`'s `url` and that of a `Fetch` on GET or HEAD, `Read`'s `path`, the final URL WebFetch's header names, Retrieve's hits, an MCP tool's declared address fields — so a tool that echoes text cannot place it under a URL nobody fetched. A result the runtime spilled to a file is read whole from that call's own spill file.
- **The run as its own source** — a body read from the harness's own state (`.crewhaus/`, a relocated session root, a tenant's root), or from a copy of one, is never a source: a session log holds every answer verbatim. A claim grounded in text an earlier call of the run wrote to that file or URL still grounds — the file does say it now — but its binding names the writer (`writtenBy`), the bundle counts it (`selfWritten`), and every surface shows it.
- **The vacuous-pass backstop** (`vacuous.ts`) — an answer the extractor finds no claim in passes vacuously only when a far blunter segmenter finds, in what a reader sees, no citation marker and no figure outside a question. Any segment that has one is verified as a claim. A greeting, a clarifying question or an answer of code alone stays vacuous; a claim run into a question (`30 days [1] — anything else?`) does not.
- **The proof bundle** (`types.ts`) — `ProofBundle`: the floor, the verdict (`pass`, `fail`, or `unverified` when the verifier threw and the gate failed open), the score and threshold, `achieved` (the highest floor the answer would have passed), each claim with its bindings, excerpts frozen at 240 characters at most with the digest of the body they came from — so a bundle stays checkable after the transcript is evicted — the assumption, hypothesis and self-written counts, and a `bundleHash` over the canonical JSON. `unverifiedBundle` is the fail-open record; `summarizeBundle` is the payload of the `provability_graded` trace line and the `provability_evaluated` audit record.
- **Rendering** (`render.ts`) — `renderFooter` (the one-line footer), `renderBundleJson`, `renderMarkdownSection` (the research report's `## Provability`) and `renderNudge`, the retry message: the model's own sentences, the identifiers it cited and plain-words reasons, never a fetched body, an excerpt or judge text, so it crosses no trust boundary. The one checker output it repeats is a rejected proof's reason and counterexample (design §0, scenario C).
- **Storage** (`store.ts`) — bundles at `.crewhaus/proofs/<spec>/<sessionId>/turn-<n>.json`, a retried turn's later passes beside the first as `turn-<n>.<k>.json`, and proof artifacts at `.crewhaus/proofs/<spec>/artifacts/<prf_id>.json`. Files are mode 0600, written atomically through `@crewhaus/tool-safety/fs` so a link planted at the name is never followed; every id is validated before it becomes a path; an artifact is never replaced; bundles carry no TTL (`crewhaus provability prune` manages size).
- **Purity** (`isPureTool`) — the one definition of which tools replay may re-run: read-only, concurrency-safe, not destructive, no sandbox, internal scope, no declared network or process capability. `Bash`, `Python`, every fetch and every write fail it.

Does not own: the text layer (`claim-ledger` — nothing here re-implements a claim, a marker, a number or a quotation); the gate's seam, ordering and `on_fail` lifecycle (`runtime-core`); building the gate's closures, the HEAD resolver, replay and the waiver judge (`evaluation-service`); the checkers (`proof-checker`); the tools (`tool-proof`); the bundle's wire schema (a strict zod mirror in `gateway-protocol`'s `proof.ts`, held to this package's type by its test); and the CLI family (`crewhaus provability show | check | audit | checkers | prune`, in `apps/cli`).

## Inputs and Outputs

Inputs: the answer text; the lowered `IrProvability`, which the compiler resolves fully at lower time so nothing here re-derives a default; an `EvidenceIndex`; a proof-artifact store; and the optional injected callbacks (`replay`, `resolveHead`, `judgeWaiver`, `recheckProof`).

Outputs: a `ProofBundle`, its rendered forms, and the persisted bundle. `verifyDocument` never throws for anything a document or its evidence can contain — an unreadable artifact, a failing judge, replay or HEAD each become a binding's reason — and throws only for a malformed config.

## Dependency Notes

Depends on `@crewhaus/claim-ledger`, `@crewhaus/event-log` (the log reader), `@crewhaus/ir` (the config type), `@crewhaus/tool-catalog` and `@crewhaus/tool-categories` (builtin names and tool schemas), `@crewhaus/tool-safety` (the atomic, link-refusing writes) and `@crewhaus/errors`. `private-address.ts` is a byte-identical copy of the repo's synchronised private-address classifier, swept and executed by a test in `apps/cli`; it is never edited here.

Consumers: `evaluation-service`, `tool-proof`, `report-writer` (the research report gate), `grader-provability`, `eval-runner` and `eval-grader` (the bundle on `RunArtifacts.proof`), `harness-advice`, `runtime-core` (the bundle type and its pure helpers) and `apps/cli`.

## First Implementation Slice

Built in the release's first stage: the evidence index, all four rung verifiers behind `verifyDocument`, the bundle and its hash, the renderers and the stores, with replay, HEAD and the waiver judge as injected callbacks that `evaluation-service` supplied in the second stage. The reviews that followed hardened the grounding rules — origins read only where a result is that origin's content, the state-root refusal, the run-as-its-own-source mark and the vacuous-pass backstop — with each shape that had slipped through pinned by a test.

## Study References

`tool-verify`'s FactCrossCheck (the grounding kernel, now claim-ledger's), `continuity-store`'s `resolveEvidence` and `FrozenProof` (the log-pairing rule and the frozen excerpt), `twelve.answerFaithfulness` (grounding is faithfulness, not truth), and design §2, §5.2–§5.4, §6 and §10 — [design/0.8.0-provability-release.md](https://github.com/crewhaus/factory/blob/main/design/0.8.0-provability-release.md).

## Validation Plan

Catalog tests: T1, T8. Primary risks: **an answer served ✓ on evidence the run never had** — a fabricated citation, a hand-computed total, a proof of a different statement — and **evidence the model wrote itself posing as a source**. The design's three acceptance scenarios are pinned with the model misbehaving; a test asserts that every `UnmetReason` is produced by some realistic input; and dedicated suites cover forged and self-written evidence and the vacuous-pass backstop.

Residuals, named rather than closed: text written by `Bash` or `RunCommand` (a command line names no place the gate can read), a write the gate cannot map to the cited origin, an earlier session's writes, and the formalization gap at `proven`.

Definition of done: `bun test packages/provability/src` green, the package importing no model adapter (the doctor finding), and `bundleHash` stable for the same answer, evidence and clock.
