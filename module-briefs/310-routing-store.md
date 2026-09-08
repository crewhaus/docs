# routing-store

Status: implemented and tested
Dependency phase: 3 - Model & Tool primitives
Catalog layer: R2 - Model Layer
Origin in ordering: Section 17 multi-provider model layer — adaptive `model_pool` routing (v0.2.2); brief back-filled with the v0.6.0 model-plan release (factory PR #440)
Workspace home: packages/routing-store
Targets: All shapes that declare `agent.model_pool` (and, since v0.4.0, per workflow step / graph node / crew role)
Test layers: T1

## Purpose

The durable reward scoreboard that makes `model_pool` routing improve with use. A pure `computeReward` maps one observed model call to a scalar; `openScoreboard` persists per-arm aggregates so the `learned` policy exploits what actually worked in this harness rather than what a table predicted.

Zero dependencies, on purpose. Everything above it — the router, the CLI's `route` verbs, the Hangar models tab — reads through this one store rather than parsing the file itself.

## Boundaries

Owns:

- **The line format.** An append-only JSONL at `<root>/routing/arms.jsonl`, mode 0600. A line is either a delta observation (one recorded model call) or an aggregate snapshot written by `compact()`. v0.6.0 adds optional `v:2` fields to a delta — judged quality, stage, strategy, attribution, would-pass, policy version, scope, harness and the arm's profile-lineage fingerprint. Every one is additive: a 0.5.x reader folds a v2 line as a plain delta, and a store that never recorded quality compacts byte-identically.
- **Arm identity.** The arm id is the `models:` profile name when the candidate is a profile, else its spec model string. Unprofiled arms keep their key, so no migration is needed; profiled arms are simply new. `crewhaus upgrade --hoist-models --rewrite-arms` re-keys the history when a hoist turns a model into a profile, and reports a reset rather than orphaning lines it cannot attribute.
- **Scoped keys.** A route key is `<scope>/<band>` with backoff to the bare band, so one store can serve several harnesses or environments without their histories fusing.
- **Lineage.** Opened with a lineage map (arm → fingerprint of the profile, quality source and judge identity), the store skips lines stamped with a fingerprint that arm no longer has: history from a profile that has since changed does not silently justify today's routing. `reward.reset_on_profile_change: false` folds it anyway; lines with no fingerprint are always kept.
- **The observe-only lanes.** A `q:` prefix (the offline quality join) and a `shadow:` prefix (the online audition) are namespaces the live router never mints and never reads, so recording them observes routing quality without steering it. `promoteLanes` is the one sanctioned fold, and `route promote` is its only caller.
- **Priors and the freeze marker.** `priors.json` seeds cold arms from an eval measurement, fingerprinted against the roster so a roster edit invalidates it loudly; `freeze.json` pins a policy version, after which pooled runs route off the recorded history and record nothing new.

Does not own: the routing decision (`model-router`'s policy router), the reward *objective* declared in the spec, or any judgement about whether an arm should be promoted — that gate lives in the CLI.

## Inputs and Outputs

Inputs: a `.crewhaus` root, route observations (`{ routeKey, model, success, latencyMs, costUsd?, quality? }`), and optional lineage and freeze context.

Outputs: `ArmStats` per arm — count, mean reward and variance, mean latency and cost, and the judged-quality mean and variance — plus lane readers, the promotion fold, and the freeze marker's read/write pair.

## Dependency Notes

No package dependencies. Append-only plus load-time replay is what makes it correct under concurrent harness processes: every run only appends its own observations with atomic small-line writes and never rewrites another run's data, so two harnesses learning into the same store cannot lose each other's updates. Aggregates fold in memory with Welford's algorithm on load; `compact()` is an explicit single-writer maintenance op.

## First Implementation Slice

Shipped with adaptive routing in v0.2.2 as the `{v:1}` delta/aggregate store. The v0.6.0 release added the `v:2` quality and attribution fields, scoped keys with backoff, lineage skipping, the two observe-only lanes with their promotion fold, the priors file and the freeze marker (factory PR #440).

## Study References

`model-router`'s policy router (the consumer that reads the arms), `watchme-store`'s quality join (which produces `q:` lane rows), and `dream-engine`'s lock discipline (the single-writer maintenance pattern `compact()` follows).

## Validation Plan

Catalog tests: T1. Primary risks: **a lost update** under concurrent harnesses (closed by append-only writes and load-time replay) and **a silently wrong arm** — history credited to a profile that has since changed, which lineage fingerprinting closes.

Definition of done: tests are green, a v2 line folds correctly on both a 0.5.x and a 0.6.0 reader, `compact()` round-trips every carried field, and the observe-only prefixes are never minted or read by the live router.
