# response-feedback

Status: implemented and tested
Dependency phase: 8 - Eval & Optimization
Catalog layer: R15 - Telemetry, Tracing, Eval
Origin in ordering: response-ratings feature (crewhaus 0.1.8, 2026-07-01)
Workspace home: apps/cli/src/feedback.ts (pure core) + apps/cli/src/index.ts (capture/distill subcommands)
Targets: cli and channel (the interactive shapes that carry the spec `feedback:` block)
Test layers: T1

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

Shipped complete in crewhaus 0.1.8 (factory PR #255): `crewhaus rate` / `feedback` / `distill [--judge]`, `optimize --ratings <session>|all` (inline distillation; synthesized grader used only when `--graders` is omitted), the cross-cutting spec `feedback:` block lowered to `ir.feedback` (deliberately not in `OPTIMIZABLE_PATHS`), Slack 👍/👎 reactions → `user_feedback` (channel/user session modes; thread no-ops), the web-UI rating bar (`@crewhaus/ui` 0.1.3), and the `response_rated` trace event.

## Study References

DSPy (program-layer optimization the distilled artifacts feed); RLHF-style preference collection (the modality set: binary/stars/scale/comment); [Recipe 56 — Response Ratings](https://github.com/crewhaus/demos/blob/main/walkthroughs/56-response-ratings.md) (the user-facing companion walkthrough).
