# response-feedback

Status: implemented and tested; the pure core has since MOVED to `@crewhaus/feedback-distill` (brief 300)
Dependency phase: 8 - Eval & Optimization
Catalog layer: R15 - Telemetry, Tracing, Eval
Origin in ordering: response-ratings feature (crewhaus 0.1.8, 2026-07-01); extracted to a package in the 0.4.x evals campaign
Workspace home: packages/feedback-distill (the core; `apps/cli/src/feedback.ts` is now a one-line re-export kept so every `./feedback` importer in `apps/cli` still resolves) + apps/cli/src/index.ts (capture/distill subcommands)
Targets: cli, channel, and managed (the shapes that carry the spec `feedback:` block)
Test layers: T1

> **Read brief [300 — `feedback-distill`](300-feedback-distill.md) for the
> current contract.** This brief records the original 0.1.8 design; the
> paragraphs below are accurate about *what the core does* but describe a
> single-rater world and a CLI-only home. Multi-rater resolution,
> adjudication, ingestion-time PII redaction, the daemon janitor step, the
> human-review queue, and `target: managed` support all arrived later.

## Purpose

Turns human ratings on real assistant turns into the two artifacts the eval stack already consumes — a `Sample[]` dataset and a `graders.yaml` — so real usage signal drives `crewhaus optimize` with no optimizer changes (Pillar 2).

A `FeedbackRecord` is one rating on one turn: a thumbs vote, a 1–5 star vote, a [0,1] scale score, and/or a free-text comment or correction. Capture surfaces write it durably as a `user_feedback` event-log line (`crewhaus rate` / `crewhaus feedback`, the compiled channel bot's Slack-reaction handler) or as bare JSONL under `.crewhaus/feedback/` (the web UI's rating bar). `distill` pairs each rating with its transcript turn and emits the dataset + one synthesized grader.

## Boundaries

Owns: the `FeedbackRecord` schema (schemaVersion 1) and its validation/clipping (untrusted comment text is bounded at 8 KiB and control-stripped at ingestion); turn derivation from a session transcript (1-based user-text ordinals, skipping tool-result echoes and `synthetic: true` recovery nudges); chronological per-turn merge (a later comment-only record keeps the earlier vote); rating normalization to [0,1] (thumbs up→1/down→0, stars n→(n−1)/4, scale→(value−min)/(max−min) clamped); tag-all distillation (positives and corrections become gold samples, low-rated turns become mutation hints); grader synthesis (exactly ONE grader — `tool_call_sequence` → `contains` → non-empty-answer floor, or an `llm_judge` rubric seeded from comment themes under `--judge`).

Does not own: filesystem access (that's `apps/cli/src/index.ts`, mirroring `doctor-checks.ts`); the Slack `reaction_added` → vote normalization (`channel-adapter-slack`) or the generated reaction handler (`target-channel-bot`); the spec/IR `feedback:` block (`spec` / `ir`); the `response_rated` trace event plumbing (`trace-event-bus`, `structured-event-printer`, `otel-exporter`); the optimize loop it feeds (`eval-optimizer-orchestrator`, `prompt-optimizer*`).

## Inputs and Outputs

Inputs: parsed session-transcript events (`user_message` / `assistant_message` / `user_feedback` lines) and bare feedback records from `.crewhaus/feedback/*.jsonl`; `DistillOptions` (`minScore`, default 0.7; `judge`; `judgeModel`).

Outputs: `Sample[]` (SampleSchema-valid JSONL via `samplesToJsonl`) and a `GradersConfigObject` (GradersConfigSchema-valid YAML via `gradersConfigToYaml`, scalars JSON-encoded so arbitrary tool names / substrings stay YAML-safe), plus distill stats and non-fatal warnings.

## Dependency Notes

Everything in `feedback.ts` is pure and side-effect-free, so it is unit-testable without fixtures. The single-grader invariant exists because stacked graders hard-AND (`all(...)` min-collapse) — one 0-scoring grader zeroes a sample regardless of the others.

## First Implementation Slice

Shipped complete in crewhaus 0.1.8 (factory PR #255): `crewhaus rate` / `feedback` / `distill [--judge]`, `optimize --ratings <session>|all` (inline distillation; synthesized grader used only when `--graders` is omitted), the cross-cutting spec `feedback:` block lowered to `ir.feedback` (deliberately not in `OPTIMIZABLE_PATHS`), Slack 👍/👎 reactions → `user_feedback`, the web-UI rating bar (`@crewhaus/ui` 0.1.3), and the `response_rated` trace event.

Since then: the core moved to `@crewhaus/feedback-distill` so a compiled daemon runs the same `distill()` on its janitor clock; reaction attribution gained an outbound-ts → `(sessionId, turnNumber)` join store so a vote lands on the **exact** reacted-to turn under every session key (including `thread`, which previously no-op'd), with the join-miss case falling back to the last turn for channel/user and dropping the reaction for thread rather than guessing; multi-rater turns resolve explicitly; free text is redacted at ingestion; and `feedback:` also parses on `target: managed`. See brief 300.

## Study References

DSPy (program-layer optimization the distilled artifacts feed); RLHF-style preference collection (the modality set: binary/stars/scale/comment); [Recipe 62 — Response Ratings](https://github.com/crewhaus/demos/blob/main/walkthroughs/62-response-ratings.md) (the user-facing companion walkthrough).
