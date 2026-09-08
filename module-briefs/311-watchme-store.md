# watchme-store

Status: implemented and tested
Dependency phase: 9 - Telemetry & eval
Catalog layer: R15 - Telemetry, Tracing, Eval
Origin in ordering: the watch-me observational-learning subsystem (factory PR #341); brief back-filled with the v0.6.0 model-plan release, which added the per-stage routing join
Workspace home: packages/watchme-store
Targets: All (per-harness store plus a machine-wide registry; not embedded in a bundle)
Test layers: T1

## Purpose

Durable "watch me" state that outlives the transcript. Sessions are swept on a 30-day TTL; what a harness *learned* by watching them should not be. Three pieces: a per-harness store of redacted session digests and judge verdicts, a machine-wide registry of the harnesses being watched, and the pure quality-to-routing join behind `crewhaus watchme report --feed-routing`.

## Boundaries

Owns:

- **The per-harness store** (`openWatchmeStore`) at `<root>/watchme`. `observations.jsonl` holds append-only digests plus Welford aggregate lines from `compact()`; `judgments.jsonl` holds the budgeted phase-2 judge's verdicts, deliberately kept out of the human feedback channel so a machine signal is never mistaken for a rating; `state.json` is written tmp-then-rename. **No TTL** — the digests must stand alone, with no pointers into raw transcripts that the sweep will delete.
- **The cross-harness registry** (`openHarnessRegistry`) — one JSON document of absolute harness directories, upsert-by-dir, atomic writes, tolerant reads. `list()` prunes tolerantly: an entry whose directory has vanished is dropped from the returned list and reported, and only `register` / `deregister` rewrite the file, so a read never races a writer.
- **The quality join** (`joinQualityToArms`) — durable route decisions and delayed quality scores meet per `(session, turn)`. The decision's profile names the arm, the same id the live scoreboard keys on, so a profiled roster is reachable from the join. Since v0.6.0 the join is **per stage**, not per turn: a hybrid turn makes several routed decisions, and folding one quality onto only the first credited the whole turn to the drafting arm and made the escalation invisible. Rows are emitted under the `q:` observe-only route key, which the live router neither mints nor reads.
- **The lock.** An advisory single-writer `run.lock` in the dream-engine mould, heartbeated while held; only a lock idle past a model-phase-scale window is stolen, so a legitimately long judge pass is never evicted.

Does not own: redaction (an injected callback at every append site — this package never imports a PII detector), the reward computation (the caller folds a joined row through `routing-store`'s `computeReward`), and the decision to act on any of it.

## Inputs and Outputs

Inputs: a harness `.crewhaus` root and a global root (`CREWHAUS_WATCHME_ROOT` or `--root`), redacted observations, judge verdicts, and — for the join — route decisions plus delayed turn qualities.

Outputs: observations and aggregates, judgments, harness registry entries, and `QualityArmRow`s carrying shadow route keys for the caller to record.

## Dependency Notes

No package dependencies, by design. The concurrency model is the scoreboard's: atomic small-line appends so concurrent harness processes never lose each other's lines, with `compact()` and `setState()` as single-writer maintenance ops that land write-then-rename, so a reader never observes a torn file.

The online twin of the offline `q:` lane is `model_pool.strategy.shadow`, whose `shadow:` arms live in `routing-store`. Both lanes are observe-only until `crewhaus route promote` folds them.

## First Implementation Slice

Shipped with the watch-me subsystem in factory PR #341 (store, registry, join). The v0.6.0 model-plan release made the join per-stage and taught it profile-named arms.

## Study References

`routing-store` (the arm identity and lane conventions it must match), `dream-engine` (the lock and compaction pattern), and `session-store`'s TTL sweep — the thing this store exists to outlive.

## Validation Plan

Catalog tests: T1. Primary risks: **stale attribution** — crediting a whole turn to the arm that drafted it — and **a digest that cannot be read once the transcript is gone**.

Definition of done: tests are green, no core function imports a PII detector, digests are self-contained, and a hybrid turn's quality reaches every stage's arm.
