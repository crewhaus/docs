# harness-registry

Status: implemented and tested
Dependency phase: 18 - Deployment & studio
Catalog layer: F3 - Deployment & Operations
Origin in ordering: 0.5.0 harness manager (Hangar M1; factory PR #372)
Workspace home: packages/harness-registry
Targets: All (machine-wide, not embedded in a bundle)
Test layers: T1, T3

## Purpose

The machine-wide harness registry: one JSON file, `<registryRoot>/harnesses.json` (format **v2**), listing every registered harness on this machine with a stable `hrn_` id, user-managed groups/tags/pins/notes, configured scan roots, and flat ordered group definitions. It is what makes `crewhaus harness list` and the Hangar console possible without a daemon, a database, or a network.

The registry root is a **directory** — `CREWHAUS_REGISTRY_ROOT`, default `~/.crewhaus` — and `CREWHAUS_NO_REGISTRY=1|true` turns every write into a no-op while reads keep working.

## Boundaries

Owns:

- **The file layer.** Root created `0700`, staging file named per-writer-unique (`${path}.${pid}.${random6}.tmp`) so two concurrent writers can never share it, written `0600`, then renamed. Around that sits a read-merge-write loop with a fingerprint check (inode + `mtimeNs` + size, `"absent"` when missing) before the rename lands, up to five attempts with a short `Atomics.wait` backoff, then a final merge-based last-writer-wins pass.
- **Id stability.** `hrn_` + 16 lowercase hex, crypto-random, minted once on first registration and stable thereafter — including across renames, since `relocate` keeps the id and moves only `dir`. Ids minted for a directory stay stable within one handle even before they are persisted; normalization pre-collects the valid ids before minting so a fresh id can never collide with a later row, and an id that fails the shape regex is re-minted.
- **`dir` as THE upsert identity key.** The absolute directory is what an upsert matches on; `findEntry` accepts either an `hrn_` id or a directory and discriminates on the regex. A refresh preserves `id`, `registeredAt`, `origin` and every user-managed field while refreshing `specName`, `target`, `kind`, `watchme.share` and `remotes`. `relocate` throws when the new directory already belongs to a different entry.
- **`missingSince`, and the rule that vanished directories are NEVER silently pruned.** `list()` stamps the current ISO time when the directory is gone and clears it back to null when it returns; that stamp change (and a pending v1/id lift) is the only thing a read will persist — a registry that has never been written is not created by a read. Only an explicit `remove()` deletes a row, and it never touches the directory.
- **Additive format discipline.** Every shape carries an index signature, so fields written by a newer CLI are tolerated on read and preserved verbatim on rewrite. Documents at `v < 2`, or in the legacy watchme format, are lifted best-effort on read and the lift persists on the next write.
- **`registerHook` / `registerHarnessHook`** — the best-effort self-registration entry point used by `run`, `compile`, `eval` and `dev` (origin `run-hook`). It never throws, and a registry write must never fail the command that triggered it.
- **watchme interop.** `seedFromWatchme()` merges the legacy store once, idempotently, with origin `import`. `upsert`/`relocate` mirror **freshness only** (dir/specName/target) onto a watchme row that already exists: a hangar-side write never creates a watchme row, never deletes one, and never touches `share`/`agentId` — membership means "explicitly watched" and belongs to the watchme verbs alone, so `watchme stop --forget` stays durable. `remove()` has no mirror at all.

Does not own: on-disk harness inspection (`harness-inventory`), process state (`harness-supervisor`, which keeps everything harness-local), the CLI verb surface (`apps/cli/src/harness-cmd.ts`), or the temp-directory exclusion, which is a CLI-edge policy: under the **default** registry root a harness dir inside the OS temp directory is skipped, because fixture compiles would otherwise fill `~/.crewhaus/harnesses.json` with guaranteed-dead rows; an explicit `CREWHAUS_REGISTRY_ROOT` means the caller took control of placement and every dir registers.

## Inputs and Outputs

Inputs: an optional root override, an injected env, and upsert/relocate/remove/group/tag/pin/scan-root mutations.

Outputs: `HangarHarnessEntry[]` sorted by directory, the document's `scanRoots` and `groups`, and a `disabled` flag that drives the CLI's `note: CREWHAUS_NO_REGISTRY is set …` line.

## Dependency Notes

Depends on `@crewhaus/watchme-store` alone. `degradeOnWriteError` exists because `list()` is semantically a read: an unwritable registry root (a root-owned `~/.crewhaus`, a read-only home filesystem, a full disk) warns and returns the computed but un-persisted view rather than failing every read surface. An unreadable or garbage file yields an empty **view** and is healed only by the next write — never by a read, in case the content is hand-recoverable.

## First Implementation Slice

Shipped in factory PR #372 with the read-only console and `crewhaus harness`; the self-registration hooks landed in the same PR across `run`/`compile`/`eval`/`dev`.

## Study References

`spec-registry` (the versioned-storage precedent this deliberately is not — this file is an index of locations, not of content), `watchme-store` (the legacy registry it seeds from and mirrors into), and the standalone-harness convention, which is why the absolute directory is the identity key.

## Validation Plan

Catalog tests: T1, T3. Primary risks: **a lost user field** (a refresh that drops groups/tags/pins turns organization into churn) and **a torn write** under two concurrent managers.

Definition of done: tests are green, an upsert of an existing directory preserves every user-managed field, a v1 document lifts without losing unknown keys, `missingSince` round-trips, and the concurrent-write test proves no writer can rename another's staging file.
