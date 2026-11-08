# control-records

Status: planned
Dependency phase: 6 - State & persistence
Catalog layer: R7 - State, Sessions, Persistence
Origin in ordering: 0.9.0 ([#507](https://github.com/crewhaus/factory/issues/507), [#513](https://github.com/crewhaus/factory/issues/513))
Workspace home: packages/control-records
Targets: All (inside compiled bundles through runtime-core and the daemon control plane; in Hangar, the CLI and the resident service, mostly through `control-service`; on the Cloudflare Worker targets the run writer is a no-op, and the harness's capabilities say so)
Test layers: T1, T3, T7, T8

## Purpose

The file layer under the shared control core ([#507](https://github.com/crewhaus/factory/issues/507)). Every control fact about a harness — its runs, the receipts of commands sent to it, its deployments, whether its operator paused or disabled it — is a file inside the harness, under `.crewhaus/run/control/`; facts about the whole workspace sit under the Hangar root. Tool-approval decisions are kept elsewhere, beside the session's `approvals.jsonl`, because a session root can move. This package decides how those files are appended to, how an exactly-once fact is settled, how a large file is read from its newest end, and how a reader resumes without missing or repeating a line. No other package opens a control file directly.

Hangar, the CLI, the resident service and compiled daemons all reach these files through the same functions, so a fact has one record whichever surface acted: `crewhaus daemon pause` and the console's Pause write the same file. Every store is a plain file, so decisions, controls and recovery keep working with Hangar stopped.

| What | File (under `.crewhaus/run/control/`) | Written by |
|---|---|---|
| Runs | `runs/` | `control-service` before a run starts, the process hosting it after (for a job's root run, the supervisor, from the job's exit code), recovery if that process died |
| Command receipts | `commands/`; workspace commands under the Hangar root | `control-service` only, whichever surface asked |
| Deployments | `deployments.jsonl` | `control-service`, from the CLI and the console alike |
| Control decisions (`dec_`) | `decisions/` | `control-service` and the features that ask a person something, by compare-and-set only |
| Operator record | `operator.json` | `control-service` only |
| Standing-rule metadata | `standing-rules.jsonl` | `control-service`'s decision service, for the owner |
| Slack link requests | `link-requests.jsonl` | a channel daemon's Slack approvals |
| Approver projection | `approvers.json` | `access-control` |
| Mutating lease | `leases/mutating.json` | Hangar's job runner and the CLI's long mutating verbs |

## Boundaries

Owns:

- **The claim-and-ledger primitive.** `appendClaim(ledger, {claim, record})` creates a claim file with `O_EXCL`, which decides an exactly-once fact, then appends the record under the ledger's lock (`withFileLock`, from `infra-utils`). The lock orders appends, and the fold a compare-and-set reads first, but it never decides anything: the shared policy steals any lock older than 30 seconds, so a writer paused across a laptop sleep can wake to find another inside. A claim carries what its record needs, so if a writer dies between the two steps, the next writer to touch it completes the append. `casAppend(id, expectState, next)` puts every compare-and-set store on the same rule: a deployment's active pointer (through a generation claim, so two activations cannot both win), `dec_` decisions and inbox assignments. Workspace ledgers use the same primitive. Removing an expired claim is a writer step, never a read.
- **Run rows without a lock.** A run row is one `O_APPEND` line under 4 KiB, small enough to land whole, and each phase of a run has one writer, so recording a run takes no lock. A row may carry `sessionAt`, `cause` and `by`, inside the 4 KiB line. A fold applies only the transitions `control-contract` allows; an illegal row is kept and shown as an anomaly, never applied. `CREWHAUS_RUN_LEDGER=0` stops run rows and nothing else: admission, decisions and consume checks never read them.
- **Segments without renames.** A growing ledger (runs, receipts, decision logs) is a series of numbered segment files. A writer appends to the highest-numbered one; past a fixed size, the next writer starts the next segment with an `O_EXCL` create, and racing writers both append there. Nothing is renamed, so no writer can append to a file a reader has already sealed. Retention, a writer step, deletes whole old segments and leaves a watermark, so a reader inside a deleted range is told it has a gap.
- **Tail readers.** Lists are newest first, so readers start at a file's newest end. A torn final line (a write still landing) waits for the next read; a malformed line in the middle is reported as a gap, and reading goes on. These lists never use Hangar's `readJsonlCapped`, which reads a file from its start up to a cap.
- **Vector cursors.** A position is a vector over each source's open segments: per file, the segment, the byte offset of the next unread complete line, and a short hash of the bytes before it. Resuming re-checks the hash. A mismatch (a copied harness, a restored backup, a replaced file) is a reset; an offset below the watermark is a gap, and so is a malformed line, after which the stream continues. The vector keeps the last two segments, so a line that lands in the previous segment just after a roll is still delivered. Positions come from files, never from clocks; their opaque encoding is `control-contract`'s codec.
- **The typed stores.** Append, fold and read functions for each record in the table, so no caller parses a control file itself. A standing rule's metadata (the decision it came from, who created it, when it expires or was revoked) lives in `standing-rules.jsonl` because `settings.json`'s strict rule schema must never gain a key: one extra key would fail every run of every 0.8.x bundle on that harness. Only the plain `{type, pattern}` entry goes in `settings.json`.
- **The operator record.** `operator.json` says whether intake is paused and whether the deployment is disabled, by whom and since when, and whether the harness should be running. It is replaced atomically, never appended, and readers cache it for one second. Absent means open. Present but unreadable means **closed** (intake paused, restarts refused, the reason reported), because it is a safety control and an atomic write never leaves a torn file. No environment variable or harness file switches it off, and the SLO gate keeps its own file, so SLO recovery can never lift an operator pause.
- **`intakeVerdict(harnessDir)`.** One answer to whether new work may enter, wrapping the operator read: `{open}`, or `{closed, reasons, notice, code, retryAfter?}`. Every door asks it: lanes, loop admission, gateway admission, `queue-consumer` and `assertRunnable`. Reasons stay independent, so lifting one never lifts another. An unreadable operator record closes it. A second reason, an exhausted spend period, joins inside this package when budgets land, so no door changes.
- **The approver-projection reader.** `approvers.json` is the harness's own copy of who may decide what, for daemons and the runtime, which must never depend on Hangar being up. Absent means single-owner mode: CLI and console decisions stand as they do today, and a Slack clicker is asked to link an account first. Present means enforced. Unreadable fails closed: a Slack click is refused, and the runtime refuses the consume and asks again. The reader passes on `homeUrl` when the projection has one, so a per-daemon card can link to Home. `access-control` decides when to rewrite it and what it says.
- **Keyed digests.** A plain sha256 of a short tool input can be guessed, so the digests that tie a record to an action (a tool's input, the places it touches, a decision's binding) are HMAC-SHA256 (WebCrypto) under the harness's own key, `keys/digest.key`, a file only its OS user can read, created by the first writer that needs it. Joins between stored digests still work; rotating the key, a deliberate erasure step, leaves every digest in the harness's live records untestable against a guess.
- **The mutating lease.** Hangar runs one mutating job per harness at a time, but enforces that with an in-process set, so an `eval` started in a terminal never sees it. `leases/mutating.json` is the same rule as a file: created with `O_EXCL`, naming its holder (pid, start time, host, surface, job kind), and taken by Hangar's job runner and by the CLI's `eval`, `optimize`, `flywheel`, `dream` and `compile`. A contender waits briefly, then refuses and names the holder. Only a provably dead holder loses the lease: it uses the pid-and-start-time check `daemon.lock` already has, but not that lock's age limit, which a long `eval` would outlive.
- **`assertRunnable(cwd)`.** One answer from `intakeVerdict` for every CLI entry point. A disabled harness refuses every entry point that would run it (exit 38). A paused one refuses non-interactive runs such as `run --prompt` and `eval` (exit 39) unless `--ignore-pause`, which the run records; the interactive REPL still runs, because the person at the keyboard is the operator.
- **The content rules, applied once.** Records hold ids, keyed digests, states, actors and refs: never tool input, user text, a secret or a raw person identifier, and a run's subject only as a keyed pseudonym. The local OS user appears as the principal it is linked to, or as `local:#<keyed digest>` when it is unlinked, never as a raw `local:<user>@<host>`. Run input waits in `inputs/` and tool input in the session's `approvals.jsonl`, each with a TTL. The two files here that name people are short-lived or rewritable: a link request expires after 15 minutes, and the projection is rewritten when access changes or a person is erased. Field names avoid what the masker redacts as a credential: `environment` and `dedupeRef`, never `env` or a `…Key` suffix.
- **Placement, and when files appear.** `.crewhaus/run/control/` is already carried by `state backup` and `retire`, and the workspace's files sit under `<hangarRoot>` (default `~/.crewhaus/hangar`). These records sit under [#507](https://github.com/crewhaus/factory/issues/507)'s runtime-state floor, with its documented limits. Every path is resolved inside its root, and every write goes through `tool-safety`, so none follows a link planted at a record's name. A file appears only when a write first needs it, never at boot, by a scan or by a read, and readers take no lock.

Does not own: ids, state machines, the event envelope and the cursor's encoding (`control-contract`); what commands do — idempotency rules, execution and recovery, the operator controls, activation, the decision service, the read models and the streams served over HTTP (`control-service`); tool-approval decisions — first-decision-wins, the consume claim and the decision log in `approvals.d/` beside the session root — which `session-store` builds on these claims, locks and segments; principals, grants, links and `authorize()` (`access-control`); process lifetimes — the supervisor's own `runs.jsonl`, which records each process rather than each run, and its runfiles and `daemon.lock` (`harness-supervisor`); writing and enforcing `settings.json` rules (`permission-engine`); when each run row is written (`control-service`, `runtime-core`, `harness-supervisor` and `gateway-server`'s `./managed-runs`); and the floor, masking and the raw inspector (`runtime-core`, `hangar-server`).

## Budget periods (`./budget`)

Status: planned

A harness's spend across processes, sub-agents, evals, optimizer calls and rehearsals, per day, week or month in an IANA zone ([#516](https://github.com/crewhaus/factory/issues/516)). Spenders append content-free reserve, settle and release lines to a durable ledger, `.crewhaus/run/control/budget/<periodId>/NNNNNN.jsonl`, and a raise appends a top-up. Closed periods roll up into `budget/periods.jsonl`, and admission reads the atomic `budget-state.json`.

The fold is crash-safe: a reservation counts until it is settled or released, or until its holder is provably dead and its lease has passed, so a crash leaks nothing. An exhausted period is `intakeVerdict`'s second independent reason, and neither resume nor SLO recovery lifts it. No new dependency.

It lands with `feat(cost-tracker): spend is never a guessed $0: unknown when unpriced, reserved before each call, budgeted by period`.

## The connection store

Status: planned

Workspace connections under `<hangarRoot>/connections/` (`con_` records plus a rewritable subjects file) and the revision-guarded writer of each harness's binding file, `.crewhaus/run/control/connections.json` ([#515](https://github.com/crewhaus/factory/issues/515)). One connection lifecycle is a `MACHINES` entry shared by every kind of connection, and retire's revokers read bindings through this package's reader.

It lands with `feat(tool-project): repository connections, bounded no-exec discovery and the project record`.

## The delegation ledgers

Status: planned

The records behind decisions an owner delegates ([#517](https://github.com/crewhaus/factory/issues/517)). Nothing is delegated until an owner turns a rule on. A harness's rules sit in `.crewhaus/run/control/delegations.jsonl`, its application lines in `.crewhaus/run/control/delegation-log/NNNNNN.jsonl`. Assistant rules keep the same pair under the Hangar root. Rule rows hold content-free terms and digests, application lines hold each check's outcome and evidence refs, and both are segmented and never compacted.

Writes go through the claim-and-ledger primitive, a rule row through `casAppend`. Readers cache for one second, and the moment before an effect reads uncached, so a revocation between a check and its effect wins. These records sit behind [#507](https://github.com/crewhaus/factory/issues/507)'s runtime-state floor.

It lands with `feat(control-service): an owner can delegate a narrow action under a rule that expires, can be revoked and is checked before every use`.

## Inputs and Outputs

Inputs: a harness directory, a session root or the Hangar root; records built from `control-contract`'s ids and states, carrying refs and digests rather than content; and, for a reader, a source and an optional position.

Outputs: appended lines and claim files, one winner per claim; folds that apply only legal transitions; newest-first pages with the position they were read at; resume answers (continue, wait, gap or reset); the operator record and the projection as absent, present or unreadable; a lease, or a refusal naming its holder; and `intakeVerdict`'s answer, with `assertRunnable`'s verdict from it.

## Dependency Notes

Depends on `@crewhaus/control-contract` (brief 322: ids, state machines, the cursor codec), `@crewhaus/infra-utils` (`withFileLock`) and `@crewhaus/tool-safety`'s `./fs` subpath (contained writes that never follow a link). It is light and node-only and spawns no process, so compiled daemon bundles can import it; `control-service` (brief 325) is heavy and never enters a bundle. It moves no emitted byte: the code that writes run rows and reads the operator record lives in libraries a bundle already imports. A bundle built with 0.8.x carries none of it, so until it is recompiled it writes no run rows and cannot enforce a pause, which Hangar shows as recorded, not enforced.

Consumers: `runtime-core` (run rows, `intakeVerdict` at loop admission, the consume check), `session-store` (claims, the lock and segments under `approvals.d/`), `gateway-protocol`'s `./control` (lanes skip ticks while `intakeVerdict` is closed; a disabled daemon refuses to boot), `gateway-server` (`intakeVerdict` at managed admission, and the managed-runs library's run rows and claims), `queue-consumer` (no pulls while `intakeVerdict` is closed), `voice-runtime` (call intake, where it has an admission seam), `harness-supervisor` (no start or restart while disabled), `channel-adapter-base` (link requests and the projection, through an interface its caller injects), `access-control` (brief 323), `control-service`, `hangar-server` (the tail reader, first for its approvals inbox) and `apps/cli` (`assertRunnable` and the lease). Each declares it in its own `dependencies`: a bundle's manifest pins only what the bundle imports directly, so an undeclared import works in the workspace and breaks once published, which is why the declarations are checked on the installed tarballs, not only in the checkout.

Why a new package: `durable-state`'s dedup and budget stores are SQLite, not files a person can inspect inside the harness; `harness-supervisor`'s ledger records process lifetimes, and the runtime must not import a package that spawns processes; and one claim primitive avoids another idempotency implementation beside `idempotency-keys`, `durable-execution` and `durable-state`. The `withIdempotency` helpers in the first two check and then run; a claim decides first. Nor is a hash-chained log needed: append-only lines plus claims give the same exactly-once facts without a writer lock on the agent loop's hot path.

## First Implementation Slice

Lands right after `control-contract` and before anything writes a control record, as one large change: the claim-and-ledger primitive, segments and watermarks, the tail reader and vector cursors, keyed digests, the operator-record and projection readers, `intakeVerdict`, the mutating lease and `assertRunnable`. It changes no emitted byte and wires one caller in the same change: Hangar's approvals inbox moves to the tail reader, so a large approvals file shows its newest asks. Its tests pin six behaviours: a torn final line waits; a malformed middle line is a gap; a replaced file is a reset; an injected lock steal still leaves one claim winner; a line that lands in the previous segment after a roll is delivered; and a closed verdict names each reason, and an unreadable operator record closes it.

The other callers, and the typed stores they write, arrive with the changes that first need them: `session-store`'s first-decision-wins builds on the claims; run rows come with canonical run identity; receipts, and the lease's first holders, with `control-service`; every door, `assertRunnable` at the CLI's entry points included, asks `intakeVerdict` with the operator controls; and the cursors serve the API's lists and resumable event streams near the end of the release.

## Study References

`packages/infra-utils/src/file-lock.ts` (the lock policy a claim exists to survive); `packages/harness-supervisor/src/runfiles.ts` (`daemon.lock`, whose pid-and-start-time check the lease copies, without its age limit) and `packages/harness-supervisor/src/queue.ts` (the in-process mutex the lease carries across processes); `packages/routing-store/src/store-io.ts` (every write through `tool-safety`) and that package's `package.json` (the scaffold a new package copies); `packages/hangar-server/src/jsonl.ts` (`readJsonlCapped`, which the tail reader replaces for the approvals inbox); the `withIdempotency` helpers in `idempotency-keys` and `durable-execution` (the check-then-run pattern a claim replaces here); [HANGAR.md](../HANGAR.md#one-state-tree-two-heads) (harness state that lives inside the harness, which this widens from processes to every control fact); and [#507](https://github.com/crewhaus/factory/issues/507).

## Validation Plan

Catalog tests: T1, T3, T7, T8. Primary risks: **two writers that both believe they won** — a lock stolen while its holder's laptop slept, or two surfaces racing — answered by claims that decide while locks only order, and pinned with an injected steal; **a reader that drops or repeats a line** — a torn write, a segment roll, deleted old segments, a copied harness — answered by tail-anchored reads and positions checked against the file's own bytes, with every miss reported as a gap or a reset rather than skipped; **a safety record that fails open** — an unreadable operator record or projection reads as closed; and **a record that leaks** — no content, keyed digests, and person identifiers only in files erasure can rewrite.

Residuals, named rather than closed: `Bash` and `RunCommand` run as the same OS user and remain the documented residual. The locks and claims assume a local filesystem, so a network filesystem is not supported. Rotating the digest key breaks joins across the rotation (each side still joins), which is the point of erasing.

Definition of done: `bun test packages/control-records/src` green with the six behaviours pinned; every reader leaves what it read byte-identical; Hangar's approvals inbox shows the newest records of a file past `readJsonlCapped`'s cap; and nothing in it can start a process: its imports are `control-contract`, `infra-utils`, `tool-safety`'s `./fs` subpath (that package's root entry also exports a process spawner) and Node built-ins other than `node:child_process`, so a bundle that takes it takes no spawner.
