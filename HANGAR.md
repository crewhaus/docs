# Hangar — the harness manager

`crewhaus hangar` boots **Hangar**, a local web console over every harness
registered on this machine, and `crewhaus daemon` is its terminal twin. The
console lists your harnesses, shows what each one is doing, and starts,
stops, restarts and drains them; `crewhaus daemon` does the same things from
a shell script with no console running at all.

Both are part of the bare `crewhaus` CLI — there is nothing extra to
install. The console's server (`@crewhaus/hangar-server`) and its static UI
(`@crewhaus/hangar-ui`) are embedded in the CLI binary.

## One state tree, two heads

The console and `crewhaus daemon` are not client and server. They are two
front ends over the same package, `@crewhaus/harness-supervisor`, and over
state that lives entirely **inside each harness**:

```
<harness>/.crewhaus/run/
  daemon.json                  the singleton runfile (atomic tmp+rename, 0600)
  daemon.lock                  the start lock
  runs.jsonl                   append-only run ledger
  control-token                the control.v1 bearer, minted at daemon boot (0600)
  logs/<runId>.log             raw captured child stdout+stderr
  logs/<runId>.events.jsonl    extracted TraceEvents (scrubbed)
  logs/<runId>.cursor          the log pump's byte-exact resume point
```

Nothing about a running daemon lives in a central manager directory. That is
what makes the two heads interchangeable: a daemon started by
`crewhaus daemon start` is **adopted** by the next console boot, and a daemon
started from the console is adopted by `crewhaus daemon status` in a terminal.
The runfile is the singleton lock, the ledger is append-only (a manager
killed mid-run leaves an *open* entry, not a corrupt file), and the log pump
resumes from a byte-exact cursor, so zero loss and zero duplication are
properties of the cursor rather than of any manager staying alive.

It also means a harness directory copied to another machine carries its own
run history with it, and that `crewhaus state backup` / `crewhaus retire`
capture that history without knowing it exists.

The manager keeps pointers, never the record. Manager-global state is small:

```
<hangarRoot>/          default ~/.crewhaus/hangar
  hangar.lock          single-instance lock (pid, startedAt, port, url)
  token                the console's bearer token (0600, 64 hex chars)
  jobs.jsonl           the durable record of queued work
  settings.json        notification rules, quiet hours, the read-only toggle
  cache/               rebuildable rollups — safe to delete wholesale
<registryRoot>/        default ~/.crewhaus
  harnesses.json       the machine-wide harness registry (format v2)
~/.crewhaus/plugins/   installed plugin manifests (always under $HOME —
                       it does not follow CREWHAUS_REGISTRY_ROOT)
```

## Five minutes to a running fleet

From a harness directory (its own `crewhaus.yaml`, per the
[standalone-harness convention](GETTING-STARTED.md#the-runtime-directory)):

```bash
crewhaus compile crewhaus.yaml -o dist   # a bundle to supervise
crewhaus harness preflight .             # will it boot? (exit 1 on blocking findings)
crewhaus daemon start                    # preflight, then spawn — supervised
crewhaus daemon status                   # runfile, liveness, control port, recent runs
crewhaus daemon logs --follow            # the scrubbed capture, tailed
```

You do not have to register anything by hand: `run`, `compile`, `eval`, and
`dev` each record the harness they touched in the machine-wide registry
(origin `run-hook`), so the list fills itself from normal use. To sweep a
directory tree in one go:

```bash
crewhaus harness scan --root ~/harnesses   # upsert every dir carrying a crewhaus.yaml
crewhaus harness list                      # what this machine knows about
```

Then open the console:

```bash
crewhaus hangar
```

It binds `127.0.0.1:4200`, prints a boxed summary, and opens your browser at
a single-use `/boot/<nonce>` URL that redirects to the token-carrying
fragment. Ctrl-C stops attached runs, **leaves daemons up** (they are
runfile-tracked and adopted by the next boot), and releases the lock.

## `crewhaus hangar`

Three verbs. Bare `crewhaus hangar` is `hangar serve` — it **boots a
server** rather than printing usage, and so does any invocation whose first
argument starts with `--` other than `--help` / `-h`
(`crewhaus hangar --port 5000` serves on 5000).

| Command | What it does |
|---|---|
| `hangar [serve]` | Boot the console and open it in the browser. |
| `hangar status [--json]` | Lock / port / registry / token report. Works with no server running — it reads the lock file, not the socket. Always exits 0. |
| `hangar open` | Re-open the running console's URL. Exits 1 when nothing is running. |

`--help` / `-h` works at the **verb position only**. `crewhaus hangar serve
--help` is an unknown-flag error, not a help screen.

### `hangar serve` flags

| Flag | Default | Effect |
|---|---|---|
| `--port <n>` | `4200` | TCP port. `0` asks the kernel for an ephemeral one. |
| `--host <h>` | `127.0.0.1` | Bind interface. Implies auth **and** requires `CREWHAUS_HANGAR_ALLOW_REMOTE=1` for any non-loopback value. |
| `--no-auth` | off | Disable the bearer token entirely. Loopback-dev only. |
| `--no-open` | off | Do not spawn the browser. |
| `--smoke` | off | Boot on an ephemeral port, run four self-checks, exit. |
| `--read-only` | off | Every mutating route answers 403. The screen-share posture; it can still be lifted from the UI. |
| `--read-only-locked` | off | As above, and the mode cannot be lifted over the wire. **Implies `--read-only`** — the lock flag alone is sufficient. |

There is no flag for the shutdown grace: children get `SHUTDOWN_GRACE_MS`
(5 s) with a `SHUTDOWN_DEADLINE_MS` (15 s) cap on confirming any one exit.

### The refusals, verbatim

`hangar serve` checks these in order, and every one exits 1 with
`crewhaus: <message>` on stderr:

```
hangar serve: unknown flag "--x" (expected: --port, --host, --no-auth, --no-open, --smoke, --read-only, --read-only-locked)
hangar serve: --port requires a value
hangar serve: unexpected argument "<x>"
hangar serve: --port must be an integer 0..65535 (got "<x>")
hangar serve: --host exposes the console beyond loopback and REQUIRES auth — drop --no-auth
hangar serve: --smoke verifies the auth surface (401 without a token) — it cannot combine with --no-auth
hangar serve: --smoke always boots on an ephemeral port — drop --port
```

The port check is deliberately strict about spelling: it rejects anything
whose canonical form differs from what you typed, so `007`, `80.0` and
`0x50` all fail.

Then the remote-bind gate, which runs **before** the lock is claimed, the
registry is opened, or a socket is bound:

```
hangar serve: --host <h> binds beyond loopback, and the console is machine control —
it starts processes, reads every transcript and writes credentials, over plain HTTP
with no TLS. Set CREWHAUS_HANGAR_ALLOW_REMOTE=1 to opt in, and put it behind a private
network (Tailscale, an SSH tunnel) — do not port-forward it.
```

And finally the single-instance lock:

```
hangar is already running at <url> (pid <n>) — use `crewhaus hangar open`, or stop that instance first
```

### What counts as loopback

Loopback is `localhost`, the whole of `127.0.0.0/8` (a daemon on `127.0.0.2`
is every bit as local), `::1` in any spelling — bracketed `[::1]`, expanded
`0:0:0:0:0:0:0:1`, or the IPv4-mapped `::ffff:127.x.x.x` — and nothing else.

**`0.0.0.0` and `::` are NOT loopback.** They are the wildcards: they bind
every interface the machine has. They are the two values that look the most
harmless, and they need the opt-in like any other remote bind.

A hostname is never resolved to decide this — DNS answers change and a name
can be pointed anywhere — so an unrecognised host fails closed. The opt-in
variable is an allow-list of exactly `1` and `true`, precisely so a typo
fails closed too.

### The boot summary

```
┌─ Hangar — the CrewHaus harness manager
│  url       http://127.0.0.1:4200/#t=<token>
│  token     ~/.crewhaus/hangar/token — sent as a URL #fragment
│  registry  ~/.crewhaus/harnesses.json (7 harness(es))
│  stop      Ctrl-C (SIGINT/SIGTERM) — stops attached runs, leaves daemons up, releases the lock
└─
```

With `--no-auth` the second row reads
`auth  DISABLED (--no-auth) — every local process can read this fleet's state`
instead, and the server logs the same warning.

Before the box, when they apply, you also get
`seeded <n> harness(es) from the legacy watchme registry`, a note about a
replaced stale lock, and
`adopted <n> running daemon(s)[, <n> gone since the last manager][, re-queued <n> pending job(s)]`.

### `hangar status --json`

```json
{
  "running": true,
  "pid": 40114,
  "port": 4200,
  "url": "http://127.0.0.1:4200",
  "startedAt": "2026-08-05T09:12:44.108Z",
  "staleLock": false,
  "hangarRoot": "/home/you/.crewhaus/hangar",
  "lockPath": "/home/you/.crewhaus/hangar/hangar.lock",
  "registryPath": "/home/you/.crewhaus/harnesses.json",
  "harnessCount": 7,
  "tokenPath": "/home/you/.crewhaus/hangar/token",
  "tokenPresent": true
}
```

`pid`, `port`, `url` and `startedAt` are present **only when running**.
`registryPath` and `harnessCount` are `null` when the registry could not be
opened. `staleLock` is true when a lock file exists but its pid is dead —
the next boot replaces it.

### `--smoke`

`hangar serve --smoke` boots on an ephemeral port, runs four checks in
order, stops the server and releases the lock. It is the release workflow's
compiled-binary smoke entry — check 2 is what proves the embedded UI assets
survived `bun build --compile`.

| # | Check | Assertion |
|---|---|---|
| 1 | `GET /healthz answers ok` | 200 and a JSON body with `ok === true` |
| 2 | `GET / serves the embedded UI shell` | 200 and the body contains `Hangar — CrewHaus` |
| 3 | `GET /api/harnesses with the bearer token answers 200 JSON` | 200 and the body parses as JSON |
| 4 | `GET /api/harnesses without a token is refused (401)` | status is exactly 401 |

Output is `smoke: booted <url> (ephemeral port <n>)`, then `✓ <name>` per
pass, then `smoke: all checks passed`. The first failure prints
`✗ <name>: <error>` and exits 1.

## `crewhaus harness`

The registry verb family. Unlike `hangar`, bare `crewhaus harness` prints
usage.

| Verb | Arguments | Flags |
|---|---|---|
| `list` | — | `--group <name>`, `--json` |
| `show` | `<dir\|hrn_id>` | `--json` |
| `add` | `<dir>` | — |
| `remove` | `<dir\|hrn_id>` | — |
| `relocate` | `<hrn_id> <newDir>` | — |
| `group` | `<name>` | `--add <dir\|id>`, `--remove <dir\|id>`, `--color <c>`, `--order <n>` |
| `tag` | `<dir\|id>` | exactly one of `--add <tag>` / `--remove <tag>` |
| `pin` | `<dir\|id>` | `--off` |
| `scan` | — | `--root <dir>` |
| `preflight` | `<dir\|id>` | `--json` |

Behaviour worth knowing:

- `add` **warns but still registers** a directory with no readable
  `crewhaus.yaml`: `⚠ no readable crewhaus.yaml in <dir> — registered anyway
  (the inventory join will show it as spec-less until one exists)`.
- `remove` drops the registry row and nothing else —
  `removed <name> (<id>) from the registry — the directory itself is untouched`.
- `scan --root <dir>` **remembers** that root as a configured scan root, so
  later bare `harness scan` calls include it. Scan never prunes vanished
  rows; it counts them: `scan: <a> added, <r> refreshed, <m> missing (<n> root(s))`.
- `preflight` works against an unregistered directory, and is the only verb
  here that can exit non-zero — **1 on any blocking finding**.

Every mutating verb appends, when writes are disabled:
`note: CREWHAUS_NO_REGISTRY is set — registry writes are disabled, nothing was persisted`.

### `harness preflight`

The typed will-it-boot report, rendered in the `doctor` check-list style:
`✗` blocking, `⚠` warn, `✓` info, with `    fix: <remediation>` under the
non-info items, and a footer of either
`<n> check(s), no blocking findings — the harness should boot` or
`<n> check(s), <b> blocking — fix the ✗ items before spawning`.

Checks run in composition order over seven areas: **spec** → **credentials**
→ **channels** → **mcp** → **ports** → **bundle** → **durability**. Each
item carries a stable `id`, so a manager can acknowledge or deep-link an
individual finding.

### `harness` vs `fleet`

Both are supported indefinitely; neither replaces the other. They are
different entry points, not a migration.

| | `crewhaus harness` | `crewhaus fleet` |
|---|---|---|
| Source of truth | **Registry-centric** — `<registryRoot>/harnesses.json` | **Filesystem-centric** — walks `--root` |
| Scope | machine-wide, wherever the directories live | whatever is under one directory right now |
| Registration | required (self-registered by `run`/`compile`/`eval`/`dev`) | none needed |
| Verbs | `list show add remove relocate group tag pin scan preflight` | `list`, `status`, `run <sub>` |
| Console | backs `crewhaus hangar` | none |

The one place they meet is `fleet --group <name>`, which reads group
membership from the machine registry — that is, from `crewhaus harness
group`. See [Fleet, lifecycle, and marketplace](CLI-REFERENCE.md#fleet-lifecycle-and-marketplace)
for the `fleet` side.

## `crewhaus daemon`

Supervise one harness from the terminal. Every verb takes an optional first
positional `[<dir|hrn_id>]`; with none, the harness is the current
directory. Bare `crewhaus daemon` prints usage.

| Verb | Flags | Notes |
|---|---|---|
| `start` | `--force`, `--ack <id,id>`, `--no-preflight` | Preflight, then spawn. |
| `restart` | same three | `stop` then `start`; the spawn plan is rebuilt, so a recompile in between is picked up. |
| `stop` | `--grace <ms>` (default `15000`) | SIGTERM, then SIGKILL after the grace. |
| `status` | `--json` | Runfile / liveness / control port / recent runs. |
| `logs` | `--tail <n>` (default `40`), `--follow`, `--run <run_id>` | The **scrubbed** capture. |
| `wake` | `--lane heartbeat\|schedule` (required), `--reason <r>` | One synthetic tick through `crewhaus.control.v1`. |
| `drain` | — | Stop intake, finish in-flight work, exit 0. |

`--grace` is accepted by `stop` but is not listed in the built-in usage
text. As with `hangar`, `--help` works only at the verb position.

Resolution errors:

```
no registered harness matches "<ref>" — see `crewhaus harness list`
<dir> is not a harness — no crewhaus.yaml here (run this from inside the harness, or pass a dir/hrn_ id)
```

### Preflight gates every start

Preflight runs before **every** start unless you pass `--no-preflight`, and
it runs against the **merged spawn env** — the harness `.env` chain
*underneath* `process.env`, which is exactly the precedence
`buildSpawnPlan` gives the child. Checking `process.env` alone is the
inversion that produces "it passed preflight and then died on a missing
key".

Any `blocking` item can be waved through individually by `id` (`--ack`) or
wholesale (`--force`) — **except** items in the `channels` area. Missing
channel secrets can never be forced, because the compiled channel daemon's
own boot gate exits 2 on exactly that set; forcing past it would trade a
clear refusal for a crash loop.

A refused start exits 1 and prints:

```
preflight refused the spawn:
  ✗ <message>
      <remediation>
      (cannot be overridden — the compiled daemon exits 2 on this)

  re-run with --force to wave through the forceable items,
  or --ack <id,id> to wave specific ones through by id.
```

### Output and exit codes

**`start` / `restart`** — on success, exit 0:

```
<specName>: started run_1a2b3c4d5e6f7a8b (pid 40231)
  class daemon · state running
  logs: crewhaus daemon logs <dir> --run run_1a2b3c4d5e6f7a8b
```

Exit 1 for `<specName>: already running (pid <p>, run <r>) — the runfile is
the lock`, for a preflight refusal, and for a plan that could not be built.

**`stop`** — exit 0 for `not running (no runfile)`, for
`the recorded daemon is gone — the stale runfile was cleared`, and for
`stopped` / `stopped (SIGKILL — it ignored SIGTERM)`. Exit **1** only for
`NOT stopped — a live daemon (pid <p>) holds the runfile but was not
adopted; nothing was signalled`.

**`drain`** — asks `POST /control/v1/drain` and falls back to SIGTERM.
Prints `drained` or `stopped (SIGTERM)` (with ` — forced` appended when the
fallback had to escalate), plus
`  control.v1 unavailable (<code>): <reason>` when the control call did not
land. Exit 0 for those and for `not running`; exit **1** only for the same
un-adopted case `stop` has — `NOT drained — a live daemon (pid <p>) holds
the runfile but was not adopted; nothing was signalled`.

**`wake`** — on success:

```
<specName>: heartbeat tick accepted (session sess_…)
  the tick runs down the daemon's OWN timer path — the same code the schedule fires
```

A refusal prints `<code>: <reason>` and exits **0 when the refusal is a
fact about the bundle** (`no_control_port`, `lane_not_armed`, `draining`)
and 1 otherwise — a bundle that armed no schedule lane is not an error, it
is an answer.

**`logs`** always renders the scrubbed capture, never the raw
`logs/<runId>.log`, which is unscrubbed by construction. Non-follow output
opens with `— <runId> —`; follow mode opens with
`— <runId> (following; Ctrl-C to stop) —`, polls every 500 ms, and exits
when the runfile's pid is no longer alive — with the last read happening
*after* the liveness check, so a crash's final lines are not dropped.
`--tail N` may reach back at most 512 KiB. An unknown `--run` id exits 1.

### `daemon status --json`

| Key | Type |
|---|---|
| `specName` | string |
| `dir` | string (absolute) |
| `target` | string (the spec's `target:`, default `"cli"`) |
| `runClass` | `interactive` \| `daemon` \| `worker` \| `one-shot` \| `mcp-server` \| `serverless` \| `export` |
| `running` | boolean |
| `runfile` | object \| null |
| `controlPort` | number \| null |
| `recentRuns` | array — the last 5 ledger entries, newest first |
| `plan` | string \| null — the equivalent shell command |
| `planError` | string \| null |

The text form ends with `  would run: <plan>` (or `  plan: <planError>`),
which is the paste-able command the supervisor would run. The env is
deliberately omitted from it, because the env carries the control token.

### Run classes and the restart policy

A harness's `target:` decides how it is launched and whether it is
supervised at all:

| Class | Targets | Launch |
|---|---|---|
| `interactive` | `cli`, `browser` | `crewhaus run <spec>` when a `crewhaus` bin resolves (harness `node_modules/.bin` first, then PATH), else `bun dist/agent.ts`; attached, piped stdio |
| `daemon` | `channel`, `managed`, `crew`, `voice` | `bun dist/daemon.ts`, detached, stdio to the log file |
| `worker` | `batch` | `bun dist/agent.ts` with daemon-class supervision |
| `one-shot` | `workflow`, `graph`, `pipeline`, `research`, `eval`, `onchain`, `onchain-game` | `bun dist/agent.ts`; tracked as jobs, never restarted. A stale bundle is *reported* (exactly when the bundle carries a spec-hash stamp, approximately from mtimes when it does not) — nothing recompiles for you |
| `mcp-server` | `expose:` / cli-shape projections | `crewhaus serve --mcp <spec> [--sse --port N]` |
| `serverless` | cf-worker emits | not a process — the plan refuses: `<target> does not run as a process — it is a deployment artifact` |
| `export` | claude-plugin emits | not a process — `… — it is a export artifact` |

Only `daemon` and `worker` classes are restarted. A stop this manager asked
for is clean and final, whatever the child reported. A supervised shape that
exits `0` **on its own** is not: it is classified `exited cleanly
(unexpected)` — a channel daemon returning 0 has lost its listener — and it
goes down the same backoff-and-window path a crash does.

These exit codes are **terminal** — never auto-restarted, because a restart
cannot fix them:

| Code | Meaning | Why no restart |
|---|---|---|
| 20 | `spec` | restarting recompiles the same spec |
| 21 | `config` | a missing key stays missing |
| 30 | `auth` | the provider rejected the credential |
| 31 | `billing` | the account is out of funding |
| 33 | `crewhaus_budget` | a restart re-arms the spend the cap just stopped |

Exit 36 (`approval_pending`) is **parked**, not failed — it is waiting on a
human, resolved by `crewhaus approvals grant`. Everything else is a crash:
exponential backoff from 500 ms, doubling, capped at 30 s, with a hard limit
of **5 restarts per rolling 10-minute window**, after which the state
becomes `crash-looping` and only a manual start moves it.

The supervision states are
`stopped → preflight → starting → running → (draining) → stopped | crashed | parked | terminal | crash-looping`.

## The registry

One file: `<registryRoot>/harnesses.json`, format **v2**. The registry root
is a *directory* — `CREWHAUS_REGISTRY_ROOT`, default `~/.crewhaus`.

Each entry carries `id`, `dir`, `specName`, `target`, `origin`
(`scan | manual | run-hook | import`), `originDetail`, `registeredAt`,
`lastSeen`, `groups`, `tags`, `pinned`, `notes`, `kind` (`local | remote`),
`watchme`, `remotes`, and `missingSince`. The document also holds the
configured `scanRoots` and the flat, ordered `groups` list.

**Ids are stable.** `hrn_` plus 16 hex characters, minted once on first
registration and kept across renames — `harness relocate` keeps the id and
moves only `dir`.

**The upsert identity key is the absolute directory.** Registering the same
directory twice refreshes the row rather than creating a second one; a
refresh preserves `id`, `registeredAt`, `origin` and every user-managed
field (`groups`, `tags`, `pinned`, `notes`) while refreshing `specName`,
`target` and friends. `findEntry` accepts either an `hrn_` id or a
directory, discriminating on the id shape.

**Vanished directories are never silently pruned.** A `list()` that finds
`dir` gone stamps `missingSince` with the current time; a `list()` that
finds it back clears the stamp. Only an explicit `harness remove` deletes a
row, and it never touches the directory. That stamp change — and a pending
format lift — are the only things a read will persist, and a registry that
has never been written is not created by a read.

**Writes are atomic and concurrency-safe.** The root is created `0700`, the
file is staged under a per-writer-unique temp name and written `0600`, then
renamed. Around that, a read-merge-write loop compares a fingerprint (inode
+ mtime + size) before the rename lands, retrying up to five times with a
short backoff before falling back to a merge-based last-writer-wins pass.
An unreadable or garbage file yields an empty *view* and is healed only by
the next write, never by a read, in case the content is hand-recoverable.

**Format discipline is additive.** Every shape carries an index signature,
so fields written by a newer CLI are tolerated on read and preserved
verbatim on rewrite. Documents older than v2 (including the legacy
watchme-format registry) are lifted best-effort on read, and the lift
persists on the next write.

### Self-registration, and what it skips

`run`, `compile`, `eval` and `dev` register the harness they touch with
origin `run-hook`. The hook is best-effort by contract: it never throws, and
a registry failure never fails the command that triggered it.

`run` and `eval` register the cwd. **`compile` and `dev` register the spec's
directory instead**, because both are routinely invoked from outside the
harness (`crewhaus compile path/to/crewhaus.yaml -o …`) and registering the
invoker's cwd would add a non-harness row whose `specName` churns to
whatever compiled last. The path is `realpath`'d first, so a spec reached
through a symlinked spelling cannot mint a second row for the same harness.

**Under the default registry root only**, a harness directory inside the OS
temp directory is skipped — fixture compiles and scratch runs would
otherwise fill `~/.crewhaus/harnesses.json` with guaranteed-dead rows the
registry never auto-prunes. Setting `CREWHAUS_REGISTRY_ROOT` explicitly
means you took control of registry placement, and then every directory
registers.

### Environment

| Variable | Default | Effect |
|---|---|---|
| `CREWHAUS_REGISTRY_ROOT` | `~/.crewhaus` | The **directory** holding `harnesses.json`. |
| `CREWHAUS_HANGAR_ROOT` | `<registryRoot>/hangar` | Where `hangar.lock`, `token`, `jobs.jsonl` and `cache/` live. |
| `CREWHAUS_NO_REGISTRY` | unset | `1` or `true` makes every registry **write** a no-op; reads keep working. |
| `CREWHAUS_HANGAR_ALLOW_REMOTE` | unset | Exactly `1` or `true` permits `--host <non-loopback>`. Anything else, including `0`, is not opted in. |
| `CREWHAUS_WATCHME_ROOT` | `~/.crewhaus/watchme` | The legacy watchme store the console seeds from. |
| `CREWHAUS_DEMOS_DIR` | unset | The demos checkout the console's demo mode copies starters from. |
| `CREWHAUS_HARNESS_ROOT` | unset | Contributes a scan-root suggestion to onboarding. |

Four more relocate a harness's data roots and are honoured *and reported* as
overrides, so fleet aggregators fold the resolved root rather than the
default: `CREWHAUS_SESSION_DIR`, `CREWHAUS_DATASETS_DIR`,
`CREWHAUS_WATCHME_ROOT`, `CREWHAUS_SHARED_DIR`.

Spawn-env precedence is: the harness `.env` chain, **overwritten by**
`process.env`, then `CREWHAUS_TRACE=json` and `CREWHAUS_COST_TRACKING=1`,
then the caller's extras (ports, the control port). An exported shell
variable always beats a `.env` file.

## `crewhaus.control.v1`

### Why it exists

A compiled daemon's schedulers are in-process — `setInterval` for
`heartbeat:`, an armed timer for `schedule:`. From outside the process there
is no way to ask "when does the next heartbeat fire?" and no way to make one
fire now: the phase of a heartbeat is knowable only inside the process that
armed it. Signals cannot carry that question, and on Windows there are no
signal semantics at all.

So every daemon-emitting target (`channel`, `managed`, `crew`, `voice`,
`batch`) now emits the same small, signal-free HTTP control plane, written
once so the five shapes cannot drift.

### Routes

All under `/control/v1`, on a loopback listener:

| Route | Answers |
|---|---|
| `GET /control/v1/healthz` | `{ok, name, target}` |
| `GET /control/v1/status` | `protocol`, `name`, `target`, `pid`, `startedAt`, `draining`, `counters{turns, heartbeatTicks, scheduleWakes, janitorRuns}`, `timers[]`, `channels[]`, `pendingApprovals` |
| `POST /control/v1/wake` | **202** `{sessionId?, lane, reason, synthetic: true}` — one synthetic tick down the identical code path the timer fires |
| `POST /control/v1/drain` | **202** `{draining: true, alreadyDraining}` |

`wake` answers 202 without awaiting the tick. `sessionId` appears only when
the lane actually threads one — on the managed and batch fan-out shapes it
does not, and advertising an id there would promise a transcript that never
existed.

Separately and independently of the control port, the channel and managed
daemons answer a bare **unauthenticated `GET /healthz` on their public
port** (deployment scaffolds have always declared that health check), and
once draining they shed every other request on that port with `503` +
`Retry-After: 15` so a PaaS backs off instead of reaping a process
mid-turn.

### Opt-in: no port, no listener

The control plane binds **only** when `CREWHAUS_CONTROL_PORT` is set and
parses as a valid port. Unset or empty means no socket at all, so upgrading
a bundle never opens a listener nobody asked for. An invalid value prints
`[control] CREWHAUS_CONTROL_PORT="…" is not a valid port — control.v1 not served`
and still binds nothing. `0` asks the kernel for an ephemeral port.
`CREWHAUS_CONTROL_BIND` (default `127.0.0.1`) moves the listener.

On bind the daemon prints, on stdout:

```
[control] crewhaus.control.v1 listening on http://127.0.0.1:53114 (token: .crewhaus/run/control-token)
```

— the port, never the token. The manager stamps `CREWHAUS_CONTROL_PORT=0`
into the spawn env and learns the real number by parsing that line, then
writes it into the runfile. A literal `0` in the runfile is treated as "not
known yet", never as a reachable port.

A deployed bundle that sets no control env vars therefore gains no listener,
no token file, and no reachable control surface — and on the two shapes that
bind a public port at all (`channel`, `managed`) exactly one new route,
`/healthz`. `crew`, `voice` and `batch` emit no HTTP server, so they gain
nothing.

### The bearer

`CREWHAUS_CONTROL_TOKEN` wins if set. Otherwise the daemon mints a fresh
32-byte token into `<harness>/.crewhaus/run/control-token` at 0600 — and
re-asserts the mode with `chmod`, because a file left by a previous boot
would keep its old permissions.

**The manager deliberately does not stamp `CREWHAUS_CONTROL_TOKEN`.** Letting
the daemon mint its own means a token left behind by a dead daemon cannot
authenticate against its replacement. The manager passes only the *path*,
records the path (never the value) in the runfile, and re-reads the file on
**every** call — a cached token 401s against its own replacement.

Server-side the comparison is constant-time over sha256 digests, a 401
carries `WWW-Authenticate: Bearer realm="crewhaus.control.v1"`, and the
rejected attempt is itself audited without the presented credential in the
record. Every call appends a `gateway_request` record to the harness's
hash-chained audit log when one is wired; an unwritable audit log never
takes the control plane down.

### Refusal codes

| `code` | HTTP | `retryable` | `expected` | Meaning |
|---|---|---|---|---|
| `no_control_port` | — | no | **yes** | The daemon is not running, or its bundle predates control.v1. |
| `no_control_token` | — | no | no | `control-token` not minted yet — it is written when the port binds. |
| `unreachable` | — | no | no | The control plane did not answer on the recorded port. |
| `lane_not_armed` | 404 | no | **yes** | This bundle armed no such lane — the spec declares no such schedule. |
| `tick_in_flight` | 409 | **yes** | no | A tick on that lane is already running. |
| `draining` | 409 | no | **yes** | The daemon accepts no new work. |
| `unauthorized` | 401 | no | no | The token on disk was refused. |
| `error` | upstream | no | no | The daemon's own error. |

`tick_in_flight` is the only retryable code. The two 409s are kept apart on
purpose: `tick_in_flight` means "come back in a second", `draining` means
"this daemon is going away" — collapsing them into one "conflict" would tell
an operator to retry into a process that is exiting.

`expected` marks the three that are *facts* rather than failures. The
console renders those as disabled-with-reason instead of an error toast, and
the CLI exits 0 on them.

### The version gate

`crewhaus daemon` will not tell you to "try again in a moment" when the
bundle can never answer. It reads the provenance stamp from the compiled
bundle's `package.json` (`dist/`, then `build/`), and compares
`compiledWith` against `0.5.0`, the first release whose emitters carry a
control plane. Pre-release suffixes are stripped, so `0.5.0-rc.1` already
counts. A stamp is trusted only on a manifest CrewHaus itself wrote, so a
hand-authored or cf-worker `package.json` reads as "no answer available",
never as "stale" — and an **unstamped** bundle is older than the control
plane by construction.

The two answers you can get:

```
no control port recorded yet — the daemon binds one at boot and announces it on stdout; try again in a moment
this bundle predates crewhaus.control.v1 (compiled with <v> | the bundle carries no provenance stamp) — recompile it with `crewhaus compile` to enable wake/drain
```

## Security model

**The bearer token is the boundary.** Every `/api` route requires
`Authorization: Bearer <token>`. Exactly three surfaces are unauthenticated:
`GET /healthz`, the `/boot/<nonce>` handoff, and the static UI shell and its
assets.

**The token file.** `<hangarRoot>/token`, 32 random bytes as 64 hex
characters, directory `0700`, file `0600`. It is **reused across boots** —
deliberately unlike the daemon's control token, because the console's token
is what you paste back into a browser tab. Comparison is `timingSafeEqual`
over sha256 digests of both sides, so unequal lengths compare in the same
time as equal ones and neither length nor prefix leaks.

**No cookies anywhere**, hence no CSRF surface. The browser keeps the token
in `sessionStorage`.

**The token never enters argv.** The URL printed in *your* terminal carries
`/#t=<token>`, but the URL handed to the browser opener is a single-use
`/boot/<nonce>` path. `argv` is world-readable on a shared machine; a
scraped nonce is already spent. The nonce is 256 bits of CSPRNG, lives at
most 120 s, and is **deleted before its expiry is checked** — spent even
when expired. The redirect answers `302` to `/#t=<token>` with
`cache-control: no-store` and `referrer-policy: no-referrer`. A fragment,
not a query string, because fragments are never sent to a server and so
cannot land in access logs or referrer headers. The browser strips it with
`history.replaceState` on arrival.

`crewhaus hangar open` follows the same rule: it reads the token off disk
and **trades** it for a fresh ticket via `POST /api/boot-ticket`, falling
back to opening the bare URL when that fails.

**Loopback by default.** `127.0.0.1:4200`, with the explicit opt-in
described above for anything else. The console is machine control over
plain HTTP: no TLS, no origin pinning, no rate limiting. That is a fine
posture for loopback and a poor one for a LAN.

**Read-only mode is an accident-guard, not an auth boundary.** The console
says so itself, in the body of `GET /api/read-only`:

> read-only mode prevents accidents during a demo or screen-share; the
> bearer token, not this toggle, is the security boundary

What it actually does: `GET`, `HEAD` and `OPTIONS` always pass; every other
method on every other path answers **403**. The exemptions are an exact set
of two — `PUT /api/read-only` (a lock nobody can lift is a support ticket)
and `POST /api/boot-ticket` (mints a browser handoff, touches no state).
They are exact strings rather than prefixes, because an exemption that
matched a prefix would quietly widen every time a route was added
underneath it. The check sits **after auth and before dispatch**, so a route
added next month is covered by construction rather than by remembering.

```json
{
  "error": "read-only mode is engaged — this manager refuses every mutating request",
  "code": "read_only",
  "method": "POST",
  "path": "/api/h/hrn_.../proc/start",
  "locked": false,
  "remedy": "turn it off in Settings (or PUT /api/read-only {\"enabled\":false})"
}
```

With `--read-only-locked`, the remedy instead reads `this manager was
started with --read-only-locked; restart it without that flag`, and a
`PUT {"enabled": false}` answers **409**. Note two honest limits: read-only
does not restrict **reads** at all (transcripts, spec, env-key presence and
costs all still stream), and boot flags are deliberately **not persisted** —
`--read-only` is a posture for one demo, and a normal restart afterwards
must give you a writable console back rather than a mystery. An explicit
API toggle *is* persisted.

**Reads are masked and contained.** Every harness payload the console
serves — transcripts, spec YAML, memory facts and their tags, wiki article
bodies, continuity focus/goals/plans, durable session summaries — passes the
same credential masker, and every file read is realpath-contained to the
harness directory, per file. The raw `logs/<runId>.log` is excluded from the
generic inspector entirely: it is unscrubbed by construction (the scrubber
sits on the supervisor's *read* path), so serving it would hand out in
cleartext exactly what the run console masks. The inspector also excludes
`.crewhaus/secrets/`, the raw audit files, and `.env` / `.env.*`, matched
case-folded because darwin and NTFS resolve `.crewhaus/AUDIT`.

**What the log scrubber does.** Any harness-env value of 8 characters or
more is replaced with `«NAME»` in captured output, except a short allow-list
of non-secret keys (`PATH`, `HOME`, `PWD`, `SHELL`, `TERM`, `TMPDIR`,
`LANG`, `USER`, `LOGNAME`, `NODE_ENV`, `LOG_LEVEL`, `PORT`, `HOST`, and the
`CREWHAUS_*` knobs that are configuration rather than credentials). A
manager adopting a *running* daemon rebuilds the scrubber from the names
recorded in the runfile — names, never values — so a name whose value the
adopting manager does not hold simply cannot be scrubbed. That is a known
best-effort edge, not a guarantee.

## What is in the console

The console is a hash-router SPA with no build step, no framework and no npm
dependency — plain ES modules, embedded as text so the compiled CLI binary
carries them.

**The fleet view** (`#/`) is a dense sortable table: name and directory
tail, shape badge, model, supervision state with parked-approval count, eval
health, session count, 7-day spend, capability badges and group chips, plus
stored groups and client-computed smart groups (Failing evals, Unbudgeted,
Has Thredz, Recently active, Ungrouped, Missing). Missing directories get a
card with relocate/remove — never a silent prune.

**The driving surfaces**: a Runs & daemons board (`#/runs`) with row actions
Start / Stop / Restart / Drain, each disabled *with its reason*; a live run
console per run, fed by SSE that opens with the durable replay and always
terminates with a `done` frame, so a finished run and a live one take the
same code path; a four-lane scheduler timeline (heartbeat / schedule / dream
/ janitor) merging spec-declared cadence with the phase only the daemon's
own process can report; and cross-harness Approvals and Review inboxes that
settle work through the same stores the CLI writes through.

**The detail surface** adds eight harness tabs and three fleet screens over
a frozen 178-route contract, across six areas:

1. **the spec's write side** — trust-tier table, diff interstitial, version
   pin/rollback, and the template / grader / dataset / MCP-connector builders;
2. **the memory fabric's write side** — facts with forget/sweep, continuity
   trash and restore, the wiki editor, watch-me analytics and synthesize;
3. **the eval / dataset / feedback loops** — matrix, trends, judge, graders,
   coverage, sentinel, the dataset registry, and the
   fewshot/faq/lessons/advice feeds;
4. **credentials + channels + security** — env-key *presence* (never a
   value), the fleet-wide provider matrix, channel provisioning and
   verification, egress / PII / compliance / on-chain / retention;
5. **Thredz**, proxied server-side so the workspace key never reaches the
   browser;
6. **the raw store inspectors**.

("Six areas" is the prose grouping. The machine-readable grouping the route
table and the left rail actually key on is eleven: `spec`, `memory`,
`evals`, `data`, `feedback`, `creds`, `channels`, `security`, `thredz`,
`inspect`, `runtime`.)

Two conventions hold that surface together.

**Every action shows the CLI command it runs.** The console is a front end
over verbs you can type; the one gap today is the session retention pin,
which has no CLI verb yet and says so.

**Every read answers `{present, note, verb}`** alongside its payload:

- `present` — is there anything to show;
- `note` — why it is empty, when it is (and it stays available for a caveat
  that survives presence, such as a truncated read);
- `verb` — the CLI verb that creates it.

So an empty panel tells you *why* it is empty and *which command fills it*,
rather than rendering a blank card. All 95 of the detail surface's GET
routes go through that base; a contract test asserts the server's route
table and the browser's are the same set — key, method and path — and then
drives every route against a live fixture server.

**Every write goes through the layer that already owns it**, never around
it: spec edits through `applySpecEdits` with `restrictToOptimizable` (a
human-owned path routes to `crewhaus propose` rather than being written),
env through `upsertEnvVar` (values in, presence booleans out), no hard
delete anywhere in the memory fabric, wiki writes carrying `expectedVersion`
with a first-class stale-version refusal, and a confirm → typed-confirm →
dry-run-first ladder in front of destructive verbs.

## Limits and known gaps

An honest list, all verified against the shipped code.

- **One route is unimplemented and says so.**
  `POST /api/h/:id/secrets/:name/rotate` answers **501**: rotation needs
  `@crewhaus/secrets-manager`, which the server does not depend on, and the
  CLI fallback was refused because `crewhaus secrets rotate <name> --value
  <value>` would put the secret in argv. Rotate from the terminal instead.
- **There is no manager-side action ledger.** `jobs.jsonl` records queued
  work and control.v1 calls append to the harness's own audit log, but a
  spec edit, an `env` set/unset, a session pin or an eval baseline re-pin
  leaves no manager-side trace.
- **`GET /api/h/:id/deployments` is read-only and empty today.** Nothing
  writes the file it reads; the empty state names it.
- **The dataset builder is not bundled.** `GET /api/h/:id/builders/dataset`
  always answers `present: false` and the POST refuses with
  `builder_unavailable` — the builder's state machine ships in the published
  dataset-builder package, which the manager does not bundle.
- **The template gallery is seed shapes only**; the full starter catalog
  lives with `crewhaus init`.
- **Experiments are not a live-traffic split.** CrewHaus's serving path does
  not split traffic by itself; `deploy canary` is an eval gate plus a pin
  flip, not a traffic splitter.
- **`audit verify` states its own two blind spots.** `anchorChecked: false`
  means records dropped off the *end* of the chain could not be ruled out;
  `externalAnchorChecked: false` means a rewrite that also rewrote the local
  anchor could not be ruled out. Those are designed limitations of a local
  verifier.
- **⌘K proposes, never acts.** `GET /api/search` executes nothing, and only
  four verbs are executable from the omnibox — start, stop, restart, drain.
  Anything else toasts `This build cannot run “<route>” from the omnibox`.
- **Plugins: two extension points, and "wired" is narrower than it sounds.**
  `panes` is genuinely wired — a plugin's pane document is served capped at
  256 KiB and rendered in an iframe with `sandbox="allow-scripts"` and *no*
  `allow-same-origin`, under a fail-closed CSP derived from the plugin's own
  `net` allow-list. `onTraceEvent` publishes the *eligibility set* — the
  manager never `import()`s plugin code, because it holds every harness's
  `.env` chain, so no trace event is delivered to a plugin callback inside
  the manager. Two further points are deferred past 0.5.0 with their reasons
  in the payload: `onSpecLoad` (a spec hook runs before the spec is trusted)
  and `onEvalSampleRendered` (eval samples carry model output and tool
  results).
- **Hangar ships no affordance that unlinks a memory file.** Forget is a
  supersede tombstone, continuity clear is a trash directory with restore,
  and wiki removal is an archived status. The only deleting path is
  `crewhaus retention sweep|purge` through the job queue, behind
  dry-run-first and a typed confirm.
- **The job queue accepts four kinds over HTTP** — `doctor`, `compile`,
  `eval`, `dream-run`. A body naming anything else gets
  `missing "kind" — one of: doctor, compile, eval, dream-run`. The console's
  other long-running actions submit internal jobs, which are not reachable
  from that route.
- **Every read is capped, and truncation is always reported.** Notably:
  5 000 JSONL lines or 8 MiB per file, 1 000 raw lines, 256 KiB of plain
  text, 500 memory items, 200 ledger rows, 500 audit records, 500 inspector
  entries. The SSE idle timeout is 255 s (Bun's maximum) because Bun's 10 s
  default would sever exactly the connections a quiet
  `heartbeat: every 60s` console exists to hold open.
- **Only `<hangarRoot>/cache/` is declared disposable.** It is rebuildable
  and never authoritative; deleting it wholesale is always safe. Nothing
  else in the tree carries that guarantee.

## Windows

The process layer ships a Windows adapter — PowerShell `Get-Process` and
`Get-CimInstance Win32_Process` for liveness and start time, `taskkill /T`
for the tree — and CI now **runs the `@crewhaus/harness-supervisor` and
`@crewhaus/hangar-server` suites on `windows-latest` as a gating job**,
including an integration test that spawns a real child and asks the real
adapter the three questions liveness is made of. A wrong answer to any of
them is what lets a restart spawn a second copy of a channel daemon, so it
is proved rather than asserted.

Three platform facts to plan around:

1. **`taskkill` without `/F` only closes windowed processes gracefully.**
   For a console daemon the graceful request is effectively a no-op, so a
   `crewhaus daemon stop` with no reachable control port waits out the 15 s
   grace and then hard-kills. On Windows the honest graceful path is
   `crewhaus daemon drain` — the signal-free control.v1 endpoint — with
   `taskkill` as the escalation behind it.
2. **The browser never opens.** The opener knows `open` (darwin) and
   `xdg-open` (linux) and is a silent no-op elsewhere. `crewhaus hangar` and
   `crewhaus hangar open` still print the URL; paste it yourself.
3. **No OS notifications.** The notifier has argv for `osascript` (darwin)
   and `notify-send` (linux) and nothing for Windows, because the PowerShell
   toast APIs differ by OS build. In-app deliveries and webhooks still work.

The compiled-binary Hangar smoke (`hangar serve --smoke`) runs on linux-x64
in the release workflow; the `windows-latest` release job submits to winget
and executes nothing. So the Windows *supervisor* is CI-verified; the
Windows *binary* is not smoke-tested by CI.

## Troubleshooting

**The console shows a token-paste screen.** Any 401 swaps the app for a
screen naming the token file. Read it and paste the value, or just re-run
`crewhaus hangar open`, which trades the token for a fresh single-use boot
ticket:

```bash
cat ~/.crewhaus/hangar/token
crewhaus hangar open
```

**`hangar is already running at … (pid <n>)`.** Another console holds the
lock. Either open it (`crewhaus hangar open`) or stop that process. If the
pid is genuinely dead the next boot replaces the lock automatically and
prints `note: replaced a stale hangar.lock left by pid <n> (process no
longer running)`; `crewhaus hangar status` will show `staleLock` in the
meantime. Note that `EPERM` counts as alive — a lock held by another user's
live process is not stale.

**`hangar open` says the console is not running, but `status` says it is.**
`hangar open` also requires a recorded `url` in the lock. Boot with
`crewhaus hangar` and check `crewhaus hangar status --json`.

**`daemon start` exits 1 with `<specName>: <error>` and no run id.** That is
`plan-failed` — the supervisor could not build a spawn plan. The two usual
causes are a target that is not a process at all
(`<target> does not run as a process — it is a deployment artifact`) and a
missing bundle. Compile first (`crewhaus compile crewhaus.yaml -o dist`),
then re-run. `crewhaus daemon status` prints the same message under
`plan:`.

**`daemon start` exits 1 with a preflight refusal.** Read the `fix:` lines.
Items marked `(cannot be overridden — the compiled daemon exits 2 on this)`
are missing channel secrets: neither `--force` nor `--ack` will move them,
because the daemon's own boot gate refuses the same set. Everything else can
be waved through by id with `--ack`, or wholesale with `--force`.

**`wake` or `drain` answers `no_control_port`.** Two different situations,
and the message tells you which. `…try again in a moment` means the bundle
speaks control.v1 and simply has not announced its port yet — the daemon
binds one at boot and prints it on stdout, which the log pump reads.
`…this bundle predates crewhaus.control.v1` means recompile:

```bash
crewhaus compile crewhaus.yaml -o dist
crewhaus daemon restart
```

An **unstamped** bundle reads as predating the control plane by
construction, so a bundle compiled by hand or emitted as a cf-worker will
always give you the second message.

**`lane_not_armed` (404).** The spec declares no `heartbeat:` or
`schedule:` for that lane. This is a fact, not a failure — the CLI exits 0
and the console renders it as disabled-with-reason.

**A 403 with `"code": "read_only"`.** The manager is in read-only mode. If
`locked` is false, turn it off in Settings. If it is true, this manager was
started with `--read-only-locked` and only a restart without that flag lifts
it.

**`409` with `"reason": "no-demos-checkout"` from the onboarding demo.**
Nothing is broken — the console cannot find a demos checkout to copy a
starter from. Clone [crewhaus/demos](https://github.com/crewhaus/demos) and
point `CREWHAUS_DEMOS_DIR` at it, or scaffold a harness directly with
`crewhaus init <dir>`.

**A harness shows as missing.** The directory moved or was deleted. The
registry never prunes it for you:

```bash
crewhaus harness relocate hrn_0123456789abcdef /new/path
crewhaus harness remove   hrn_0123456789abcdef   # or drop the row; the dir is untouched
```

**Registry writes appear to do nothing.** Check `CREWHAUS_NO_REGISTRY` —
when it is set, every write is a no-op and each mutating verb says so on its
last line.

## Next steps

- [CLI-REFERENCE.md — The hangar](CLI-REFERENCE.md#the-hangar) — the
  authoritative flag tables for `hangar`, `harness`, and `daemon`.
- [WEB-UI.md](WEB-UI.md) — `@crewhaus/ui`, the per-shape UI for a *single*
  compiled harness. Hangar manages the fleet; `@crewhaus/ui` renders one
  bundle's `TraceEvent` stream.
- Module briefs:
  [`hangar-server`](module-briefs/301-hangar-server.md),
  [`hangar-ui`](module-briefs/302-hangar-ui.md),
  [`harness-registry`](module-briefs/303-harness-registry.md),
  [`harness-inventory`](module-briefs/304-harness-inventory.md),
  [`harness-supervisor`](module-briefs/305-harness-supervisor.md),
  [`preflight`](module-briefs/306-preflight.md).
- Source:
  [`apps/cli/src/hangar-cmd.ts`](https://github.com/crewhaus/factory/blob/main/apps/cli/src/hangar-cmd.ts),
  [`apps/cli/src/daemon-cmd.ts`](https://github.com/crewhaus/factory/blob/main/apps/cli/src/daemon-cmd.ts),
  [`apps/cli/src/harness-cmd.ts`](https://github.com/crewhaus/factory/blob/main/apps/cli/src/harness-cmd.ts),
  and the packages under
  [`packages/`](https://github.com/crewhaus/factory/tree/main/packages).
