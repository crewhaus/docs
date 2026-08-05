# harness-supervisor

Status: implemented and tested
Dependency phase: 18 - Deployment & studio
Catalog layer: F3 - Deployment & Operations
Origin in ordering: 0.5.0 harness manager (Hangar M2; factory PRs #373/#375/#376)
Workspace home: packages/harness-supervisor
Targets: All (it supervises compiled bundles of every shape)
Test layers: T1, T3, T7

## Purpose

The process layer: spawn contracts per run class, the preflight gate, the supervision state machine with its restart window and exit classification, durable log capture with a byte-exact `TraceEvent` pump, adoption of processes it did not start, and the bounded job queue.

The design constraint that makes everything else possible: **all supervision state is harness-local**, under `<harness>/.crewhaus/run/`. Nothing lives in a central manager directory. That is what lets the Hangar console and `crewhaus daemon` drive the same daemon from either side, and what makes a harness copied to another machine carry its own run history — `state backup` and `retire` capture it without knowing it exists.

## Boundaries

Owns:

- **`runfiles.ts`** — the on-disk contract. `daemon.json` (the singleton runfile, atomic same-directory tmp + rename, 0600), `daemon.lock` (the start lock), `runs.jsonl` (append-only ledger), `logs/<runId>.{log,events.jsonl,cursor}`, `control-token`, with `run/logs` created `0700` and run ids shaped `run_<16 hex>` — deliberately the same SAFE_ID shape the server's path guards accept, so a runId can be a URL segment without further checks.
- **The runfile as a claim, not a fact.** Liveness requires **all three** of pid, OS process start time, and an `argvFingerprint` (a digest of the exact `[cwd, ...argv]`) to still match, which is why a runfile restored from a `state backup` fails by construction and reads as stale. Unreadable, unparseable and shape-invalid all read as "no runfile" — a corrupt runfile must never wedge a harness shut.
- **`daemon.lock`** — the cross-process claim on the daemon *slot* while a start is in flight, because the runfile cannot be that claim: there is no pid until the spawn, and the whole preflight run sat inside that gap, so two managers could both read "no runfile", both pass preflight, and both spawn. `openSync(path, "wx", 0o600)` is atomic across processes, so exactly one starter wins. Breakable when the holder is gone, when the pid was recycled (start time moved), or after `START_LOCK_MAX_AGE_MS = 120_000`; an unknown start time errs toward "held".
- **`runs.jsonl`** — append-only. A run is OPENED with a full record and CLOSED by appending a partial record carrying the same `runId`; readers fold by id, later wins. A manager killed mid-run therefore leaves an *open* entry, not a corrupt file, and torn lines are skipped rather than fatal. The single exception to never-rewriting is retention compaction, which rewrites the whole file atomically — safe because the supervisor is the single writer of this directory.
- **`spawn-contracts.ts`** — `RUN_CLASS_BY_TARGET` (`interactive`, `daemon`, `worker`, `one-shot`, `mcp-server`, and the two non-process classes `serverless` / `export`, whose plans refuse with `<target> does not run as a process — it is a deployment artifact`), the cwd rule (the harness root, found by walking up at most four levels), and `buildSpawnEnv`'s precedence: the harness `.env` chain, **overwritten by** `process.env`, then `CREWHAUS_TRACE=json` and `CREWHAUS_COST_TRACKING=1`, then caller extras. `OVERRIDE_ENV_KEYS` (`CREWHAUS_SESSION_DIR`, `CREWHAUS_DATASETS_DIR`, `CREWHAUS_WATCHME_ROOT`, `CREWHAUS_SHARED_DIR`) are honoured **and reported**, so a fleet aggregator folds the resolved root rather than the default.
- **`gate.ts`** — the preflight gate. Any blocking item can be acknowledged by `id` or forced wholesale, **except** `UNFORCEABLE_AREAS = {"channels"}`: missing channel secrets can never be waved through, because the compiled channel daemon's own boot gate exits 2 on exactly that set, so forcing would trade a clear refusal for a crash loop.
- **`policy.ts`** — exit classification and restart policy. Terminal codes (20 spec, 21 config, 30 auth, 31 billing, 33 crewhaus_budget) are never auto-restarted; 36 (`approval_pending`) is **parked**, not failed; a `0` from a long-running shape is clean but `unexpectedClean` and is restarted by default. Everything else backs off 500 ms → ×2 → capped 30 s, with `MAX_RESTARTS_PER_WINDOW = 5` in a rolling 10 minutes before the state becomes `crash-looping`.
- **`trace-pump.ts`** — the extracted `TraceEvent` stream, split with the balanced-JSON splitter shared byte-for-byte with the public `@crewhaus/ui` host, and a cursor that records the byte offset of the last fully consumed text **and** the events file's size at that moment. On resume the events file is truncated back to that size, so zero loss and zero duplication are properties of the cursor, not of the manager staying alive.
- **`scrub.ts`** — the credential scrubber on the **read** path. Any harness-env value ≥ `MIN_SCRUBBED_VALUE_LENGTH = 8` becomes `«NAME»`, except the `NON_SECRET_ENV_KEYS` allow-list. The raw `logs/<runId>.log` is unscrubbed by construction, which is why it is an explicit exclusion from the console's generic inspector.
- **`shutdown.ts`** and **`queue.ts`** — manager-wide shutdown (`SHUTDOWN_GRACE_MS = 5_000` per child, `SHUTDOWN_DEADLINE_MS = 15_000` cap on confirming one exit; daemons **detach** and are adopted by the next boot, everything else is stopped, running jobs are signalled and left open for `restore()` to reopen as `interrupted`), and the job queue with `DEFAULT_JOB_CONCURRENCY = 3` and the harness mutex.
- **`process-ops.ts`** — the POSIX and Windows adapters. On Windows, `terminate` is `taskkill /PID <pid> /T` and `forceKill` adds `/F`; liveness and start time come from PowerShell `Get-Process` / `Get-CimInstance Win32_Process` as a per-call `spawnSync` with a 5 s timeout.

Does not own: the checks themselves (`preflight`), the registry (`harness-registry`), the control plane's wire contract (`gateway-protocol`), or either front end.

## Inputs and Outputs

Inputs: a harness directory, an injected `Clock`, injected `ProcessOps`, an env record, and an optional preflight gate.

Outputs: a `HarnessSupervisor` (`start`, `stop`, `drain`, `adopt`, `status`), spawn plans plus their `cliTwin` (the paste-able shell equivalent, with the env deliberately omitted because the env carries the control token), run ledger entries, scrubbed log tails, and the typed gate refusal.

## Dependency Notes

Depends on `@crewhaus/errors`, `@crewhaus/harness-inventory`, and `@crewhaus/preflight`. Retention defaults to 20 runs / 50 MiB, overridable per harness under `manager.logRetention` in `<harness>/.crewhaus/settings.json` — any read or shape problem degrades to the defaults, because retention is never a reason to refuse a spawn. Pruning removes a run's `.log`, `.events.jsonl` and `.cursor` **together**; they are one artifact.

Adopted-run scrubbing is best-effort by construction: the runfile records scrub key **names, never values**, so an adopting manager rebuilds an equivalent scrubber from its own env and a name whose value it does not hold cannot be scrubbed. Dropping those names entirely is what once let a `process.env`-sourced secret land in cleartext in the durable events file after a restart.

## First Implementation Slice

Shipped in factory PR #373 (M2, with `crewhaus daemon`), extended by #375 (manager-wide shutdown and the job-queue semantics) and hardened by #376, whose first gating Windows CI run found six real bugs — none of them the predicted spawn problem, all path handling, two of them containment checks written as `startsWith(root + "/")`, which never matches `D:\a\b\c`, and one of which failed **open**.

## Study References

`daemon-process` and `scheduler` (the in-bundle precedents this supervises from outside), `@crewhaus/ui`'s `_shared/host.ts` (the identical `TraceEvent` splitter), and `crewhaus.control.v1` in `gateway-protocol`, which is the graceful path on platforms without signal semantics.

## Validation Plan

Catalog tests: T1, T3, T7, plus a Windows-only integration test that spawns one real child and asks the real adapter the three questions liveness is made of. Primary risk: **a second copy of a daemon** — a wrong liveness answer lets a restart double-process every message and double the provider spend, which is why liveness is a three-part match rather than a pid check.

Definition of done: the suite is green on POSIX **and** on `windows-latest` as a gating CI job; a runfile survives round-tripping; the ledger folds correctly with an open entry; and the pump's cursor proves zero loss and zero duplication across a simulated manager restart.
