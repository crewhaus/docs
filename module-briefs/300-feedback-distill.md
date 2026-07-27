# feedback-distill

Status: implemented and tested
Dependency phase: 8 - Eval & Optimization
Catalog layer: R15 - Telemetry, Tracing, Eval
Origin in ordering: 0.4.x evals campaign (D39/B19/B20/B23) — the package extraction of brief 291's core
Workspace home: packages/feedback-distill
Targets: EVAL, cli, channel, managed
Test layers: T1

## Purpose

The human-rating half of the improvement flywheel, in a package so **both** the toolchain (`crewhaus rate` / `feedback` / `distill`, `optimize --ratings`, the `crewhaus run` teardown) and a **compiled daemon bundle** run exactly the same code. `feedback.autoDistill` used to have one production consumer — the `crewhaus run` teardown — while the shapes that actually generate ratings are the daemons. Their feedback piled up until somebody happened to run the CLI against that harness. Extracting the core fixed that: the daemons register a janitor step and distil on their own clock.

## Boundaries

Owns:

- **`feedback.ts`** — the `FeedbackRecord` schema and its validation/clipping, turn derivation from a session transcript, rating normalization to [0,1], `distill()`, and grader synthesis (still exactly ONE grader; stacked graders hard-AND, so a second zero-scoring grader would zero the sample).
- **Multi-rater resolution.** Feedback stays append-only, so every rater's record survives. All-thumbs turns resolve by **majority**; stars/scale (or a mixture) by the **mean normalized score**; an `--adjudicate` record always wins and closes the disagreement. A true split verdict with no adjudication is **withheld** from the dataset and enqueued for review rather than silently labeled. Resolved samples carry every rater's normalized verdict in `metadata.ratings`, plus `metadata.adjudicated` when an adjudication settled it. Single-rater corpora — including everything recorded before this release — distil byte-identically.
- **`redact.ts`** — the shared PII/secret detector set, applied at sample construction so raw text never reaches the dataset, the synthesized graders, or an optimizer meta-prompt. `--no-redact` exists on `distill`, `dataset mine` and `optimize --ratings`; the unattended janitor tick has **no** opt-out.
- **`review-queue.ts`** — the append-only human-review queue at `.crewhaus/review/queue.jsonl` (`REVIEW_QUEUE_SCHEMA_VERSION = 1`). Kinds are exactly `abstained | needs_review | rater_disagreement | quarantine`; a `ReviewSourceRef` is exactly one of `{runId, sampleId}`, `{sessionId, turn}` or `{dataset, sampleId}`; status is `open | resolved`. Entry ids are deterministic from the source key, so all three feeders (`crewhaus eval` at run end, `distill`, `dataset mine`) are idempotent.
- **`watermark.ts`** — the auto-distill trigger (≥ 5 unprocessed ratings by default, `CREWHAUS_AUTODISTILL_THRESHOLD` to override), the shared watermark file and full-rebuild semantics.
- **`janitor-step.ts`** — `createDistillJanitorStep`, the `feedback_distill` step the channel and managed daemons register (`CREWHAUS_AUTODISTILL=0` disables the tick).
- **`collect.ts`** and **`split.ts`** — feedback collection from the root the *daemon* actually writes to, and the deterministic split/version helpers used when promoting a `<name>-ratings` registry version.

Does not own: filesystem policy at the CLI edge (`apps/cli/src/index.ts`); the Slack `reaction_added` → vote normalization (`channel-adapter-slack`) or the generated reaction handler and its outbound-ts join store (`target-channel-bot`); the spec/IR `feedback:` block (`spec` / `ir`); the `response_rated` trace event plumbing; the optimize loop it feeds.

## Inputs and Outputs

Inputs: parsed session-transcript events (`user_message` / `assistant_message` / `user_feedback`), bare feedback records from `.crewhaus/feedback/*.jsonl` (including the managed daemon's per-tenant files), and `DistillOptions` (`minScore`, default 0.7; `judge`; `judgeModel`).

Outputs: `Sample[]` (SampleSchema-valid JSONL), a `GradersConfigObject` (GradersConfigSchema-valid YAML), distill stats and non-fatal warnings, per-turn agreement plus an overall **Cohen's kappa** whenever any turn carries ≥ 2 raters, and review-queue entries for the verdicts a human has to settle.

## Dependency Notes

The distillation core is pure and side-effect-free — that is what made the extraction possible, and it stays unit-testable without fixtures. Base distillation is fully offline (no model call), so the credential-stripped-hooks rationale that keeps auto-distill out of most bundles does not apply to the janitor step. The watermark is a shared read-modify-write, not a lock: once one consumer lands a batch the others see nothing unprocessed. Registration happens first and the watermark advances only on success, so a transient registry failure retries next tick — and a sweep that finds no readable transcript **skips** rather than advancing, because a misconfigured session root used to look exactly like "these ratings are unmatchable" and burn them.

## First Implementation Slice

Shipped as part of the 0.4.x evals campaign: the package extraction plus the daemon janitor (D39), multi-rater resolution and adjudication (B19), the review queue and its three feeders (B20), and ingestion-time redaction (B23).

## Study References

Brief 291 (the original response-ratings design this extends); RLHF-style preference collection for the modality set; inter-rater reliability (Cohen's kappa) for the agreement reporting; [Recipe 62 — Response Ratings](https://github.com/crewhaus/demos/blob/main/walkthroughs/62-response-ratings.md).

## Validation Plan

Catalog tests: T1. Primary risks: **silently mislabeled training data** — a split verdict resolved by coin-flip, or a rating attributed to the wrong turn, both produce a dataset that looks fine and teaches the wrong thing; and **watermark burn**, advancing past ratings that were never actually distilled.

Definition of done: tests are green, public types are exported from the package entrypoint, the janitor step and the CLI teardown demonstrably run the same `distill()`, and a single-rater corpus distils byte-identically to the pre-change output.
