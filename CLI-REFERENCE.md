# CrewHaus CLI reference

The complete `crewhaus` command surface, grouped by task. This is the
navigation companion to the one-line subcommand table in
[GETTING-STARTED.md](GETTING-STARTED.md#the-cli-end-to-end); it documents
every subcommand and its key flags. The authoritative help is always
`crewhaus <subcommand> --help` (and the top-level `crewhaus --help`),
generated from the CLI at
[`apps/cli/src/index.ts`](https://github.com/crewhaus/factory/blob/main/apps/cli/src/index.ts).

You invoke the CLI from inside a harness directory (the standalone-harness
convention — spec, `.crewhaus/` state, datasets, and MCP config all resolve
from the current working directory). Install it with
[chvm](https://github.com/crewhaus/chvm), the CrewHaus version manager and
the recommended path — `npm install -g @crewhaus/chvm`, on macOS, Linux, or
Windows. Mind the scope: the bare `chvm` on npm is an unrelated package.
`chvm use
<version>|latest|system|local` points the shim at a pinned npm install, the
system `crewhaus` on your PATH, or a factory checkout run from source, and a
switch applies immediately in every open shell. Homebrew, Scoop, winget,
apt, and npm all still ship the CLI. See
[GETTING-STARTED](GETTING-STARTED.md) for every channel.

> **Everything here is additive and opt-in.** The CLI is a full lifecycle
> surface, well beyond a handful of build/run/eval verbs — but
> existing specs compile byte-identically and every automation is a default,
> an opt-out flag, or a propose-then-confirm flow over controls that still
> work by hand. Nothing in this reference removes control; the optimizer's
> hard floors (`permissions` and `model` fields stay outside
> `OPTIMIZABLE_PATHS`) are unchanged.
>
> **v0.4.0** carries the same posture into the agent-loop surface: new verbs
> (`dev`, `serve --mcp`, `runs resume`, `approvals`, `export claude-plugin`,
> `deploy <fly|render|railway|heroku>`, `failures report`, `sessions tail|export`)
> and loop-contract flags (`compile --emit-loop` / `--emit-as cf-worker` /
> `--strict`, `run --trace` / `--streaming` / `--plugins`, `eval --repeats`) are
> all additive — existing specs still compile byte-identically.
>
> **v0.4.x evals** adds the largest batch since v0.2.0: `eval suite`,
> `eval plan`, `redteam generate|report`, `review list|next|resolve`,
> `experiment status|record|assign`, `schedule generate`,
> `eval-report trends|export`, `graders test|card`,
> `datasets verify|status|release|card`, and `dataset audit|lint` — plus new
> flags across `eval`, `optimize`, `flywheel`, `eval-report diff`,
> `scaffold-evals`, `init`, `distill`, `dataset mine`, `rate`/`feedback`, and
> `deploy canary`. Three of those are **behaviour changes**, not additions,
> and are called out where they land: the judge's decoding `temperature` is
> now pinned to **0** by default; a bare `registry:<name>` dataset ref now
> resolves **train+dev only** (an explicit `#test` needs `--allow-test-split`,
> accepted by `eval` and `deploy canary` alone); and `distill` /
> `dataset mine` now PII-redact sample text by default (`--no-redact` opts
> out; the unattended `autoDistill` teardown always redacts).
>
> **v0.5.0** adds the harness manager — [`hangar`](#the-hangar),
> [`harness`](#crewhaus-harness), and [`daemon`](#crewhaus-daemon) — plus
> the `crewhaus.control.v1` surface compiled daemon bundles now serve. It is
> additive in the same way: `crewhaus fleet` is kept indefinitely and
> unchanged, existing specs compile byte-identically, and a bundle that sets
> no control env vars binds no control listener and mints no token file —
> on the two shapes that bind a public port (`channel`, `managed`) it gains
> exactly one new route, `/healthz`. See [HANGAR.md](HANGAR.md).

## Table of contents

1. [Build, run, and author](#build-run-and-author)
2. [Health and diagnostics — `doctor`](#health-and-diagnostics--doctor)
3. [The self-building eval flywheel](#the-self-building-eval-flywheel)
   — [running an eval](#running-an-eval) · [suites, planning, and
   red-teaming](#suites-planning-and-red-teaming) · [reporting and
   analysis](#reporting-and-analysis) · [optimize, flywheel, and
   scheduling](#optimize-flywheel-and-scheduling)
4. [Datasets, graders, and judges](#datasets-graders-and-judges)
   — [the dataset registry](#the-dataset-registry) · [growing and auditing a
   dataset](#growing-and-auditing-a-dataset) · [graders and
   judges](#graders-and-judges)
5. [The observer/advisor](#the-observeradvisor)
6. [Model and cost automation](#model-and-cost-automation)
7. [Feedback, memory, and knowledge](#feedback-memory-and-knowledge)
8. [Self-healing operations](#self-healing-operations)
9. [Deploy, promote, and govern](#deploy-promote-and-govern)
10. [Fleet, lifecycle, and marketplace](#fleet-lifecycle-and-marketplace)
11. [The hangar](#the-hangar)
    — [`hangar`](#crewhaus-hangar) · [`harness`](#crewhaus-harness) ·
    [`daemon`](#crewhaus-daemon)
12. [Safety that learns](#safety-that-learns)
13. [Compliance, audit, and retention](#compliance-audit-and-retention)
14. [Registry, secrets, and state](#registry-secrets-and-state)
15. [Distribution and infrastructure](#distribution-and-infrastructure)

Flag conventions in this document: `<required>` positional or value,
`[--optional]`, `a|b|c` an enum of choices, `-o` / `--out` a short alias.
Values marked `registry:<ref>` accept the
`registry:<name>[@version][#split]` shorthand for a dataset stored in the
local registry (see [`datasets`](#datasets-graders-and-judges)).

> **Test-split lock (v0.4.x behaviour change).** A *bare* `registry:<name>`
> now resolves **train+dev only** on every consuming path, printing a
> one-line stderr notice when a test split existed and was excluded. An
> explicit `#test` requires `--allow-test-split`, and exactly two verbs
> accept that flag: `eval` and `deploy canary`. `optimize` and `flywheel`
> refuse `#test` outright regardless of flags. Inspection is not
> consumption, so `datasets get`, `eval coverage`, `dataset audit`, and
> `dataset lint` still read every split (and say so). On a record that
> carries a test split, bare-ref runs therefore grade fewer samples and the
> dataset content hash changes — the next `eval` starts a new
> `(spec, dataset)` baseline lineage.

---

## Build, run, and author

The everyday loop: scaffold a spec, compile it to a bundle, run it.

| Command | Purpose |
| --- | --- |
| `init [name]` | Scaffold a fresh `crewhaus.yaml`. |
| `init --interactive [--detect]` | Interview-driven spec authoring (via the resolved model, or a scripted stdin questionnaire with no credentials); the result is validated with `parseSpec`. |
| `init --ci` | Also scaffold `.github/workflows/crewhaus-eval.yml` — an eval-gated spec-PR check. Composable: run it in an existing harness to add just the workflow. |
| `init --ci\|--sentinel --suite <suite.yaml>` | Scaffold the **tiered** workflow against a suite manifest instead: `--ci` emits fast-tier-on-PR plus a nightly-cron job for the nightly tier and a tier-verdict PR comment; `--sentinel` gives the drift cron a nightly-tier step that runs even when the probe failed. The suite path must be harness-relative and inside the harness. Without `--suite`, both scaffolds are byte-identical to before. See [the eval flywheel](#the-self-building-eval-flywheel). |
| `init --with-evals` | Also scaffold `eval/dataset.jsonl` + `eval/graders.yaml` (offline template mode — no credentials needed) so the harness can `crewhaus eval` on day one. |
| `init --sentinel` | Also scaffold `.github/workflows/sentinel-drift.yml`, the nightly model-drift sentinel cron. |
| `compile <spec> -o <out-dir>` | Parse → IR → emit a runnable bundle. **The FR-002 external-sink scope gate runs by default** — the build fails if an I/O-capable tool is left at a non-`external` scope. |
| `compile <spec> --emit-ir [-o <dir>]` | Print the lowered IR as JSON (or write `<dir>/ir.json`); skip codegen. |
| `compile <spec> --emit-loop [--json] [-o <dir>]` | Skip codegen and print the canonical **agent-loop projection** (`projectLoop` of the lowered IR) — the exact wire shape the Studio `/builder` renders and the compiler-worker's `POST /loop` returns. Human-readable by default; `--json` prints the raw `LoopProjection`; `-o` writes `<dir>/loop.json`. A read-only view: nothing is emitted and (matching `POST /loop`) the FR-002 scope gate does **not** run, so you can inspect the loop of a spec whose tool scopes still need fixing. Mutually exclusive with `--emit-ir`. |
| `compile <spec> --emit-as local\|cf-worker` | `local` (default) emits the standalone Bun bundle; `cf-worker` emits the Cloudflare-Worker bundle (`worker.js` + `wrangler.toml` + `package.json`) — the **same** bundle the Studio's remote compiler (`compiler-worker POST /compile`) serves. Supported for `target: cli\|workflow\|graph`; other shapes are rejected. Incompatible with `--emit-ir`/`--emit-loop`/`--check`/`--with-eval-harness`. |
| `compile <spec> --strict` | Escalate compile **warnings** (accepted-but-unwired spec keys) to errors — warnings always print (code + path + message, one per line), and with `--strict` any warning fails the compile before files are written. Distinct from the default-on FR-002 external-sink scope gate (which is governed solely by `--allow-unmarked-sinks`). |
| `compile <spec> --check` | After emitting, verify the bundle: shape smoke assertion → `bun install` → credential-free liveness boot. Red exits 1. |
| `compile <spec> --watch` | Re-run parse → lint → compile on every change to the spec / commands / skills dirs. Debounced, Ctrl-C-clean, one green/red status per cycle. |
| `compile <spec> --with-eval-harness [--eval-dataset <name>]` | Also emit an eval bridge — a `target: eval` bundle projected from **this** (non-cli) shape — into `<out-dir>/eval/`, so that shape can consume its distilled feedback through eval/optimize/flywheel. The bridge is **runtime-invoking**, not a single-turn impersonation: `workflow`/`graph`/`crew`/`pipeline` samples drive the shape's compiled bundle end-to-end (the primary bundle gains an exported eval entry under this flag), `channel` samples run the bot's real `runTurn` through a loopback, `managed` samples drive the gateway's `runOneTurn` dispatcher under an isolated per-sample tenant, and the remaining shapes run their agent plus real wired tools through the single-turn loop — the chosen strategy is printed per compile. Sample `history` seeds only the chat-capable shapes (`channel` / `managed` / `voice` / `pipeline`); a history-carrying sample against any other bridged shape is **rejected loudly at dataset load**. Rejected for `cli` (use `crewhaus eval` directly) and incompatible with `--emit-as cf-worker`. A plain compile stays byte-for-byte identical. |
| `lint [spec] [--fix] [--format text\|json]` | Check-only: `parseSpec` + compile with IR passes (the §47 chain / graph-crew well-formedness checks the CLI compile path skips) + scope audit, **without emitting**. `--fix` does nearest-match repairs for typo'd tool names / secret refs. `--format json` for editors and CI. |
| `run <spec> [--model <m>]` | Compile in-memory and execute the agent (cli shape). |
| `run <spec> --resume <id>` / `--continue` | Resume a specific session, or the most-recent one (cli targets). |
| `run <spec> --prompt <text>` | Run a single turn and exit, printing the reply — no REPL (cli one-shot, for scripting/CI). For browser targets it is the single-turn input; other shapes fall back to stdin. |
| `run <spec> --budget-usd <N>` | Run-level spend cap; sets/overrides the spec `budget.usd` ceiling and keeps its `on_exceed` ladder. |
| `run <spec> --trace off\|ring\|pretty\|json` | Override the spec's `observability.trace.level` for this run: `pretty`/`json` attach the structured-event printer, `ring` keeps only the bus ring buffer (default), `off` suppresses the printer. The flag wins absolutely over the spec block **and** the ambient `CREWHAUS_TRACE` env. |
| `run <spec> --streaming` | Force streaming tool dispatch on for this run — tools execute as each `tool_use` block completes mid-stream. The spec's `agent.streaming` sets the default; the flag wins. |
| `run <spec> --plugins <a,b,c>` | Override the spec's `plugins:` list for this run — a comma-separated set of installed plugin names to activate at boot. `--plugins ""` activates none. |
| `run <spec> --user <id>` | Identify the user so `.crewhaus/preferences/<user>.md` is injected at run start (alongside the auto-loaded `LESSONS.md`). |
| `run <spec> --justification-judge rule-based\|claude` | Pillar 3 intent-gate judge for this run (FR-004); overrides `security.justification.judge`. |
| `run <spec> --egress-matcher substring\|semantic [--egress-embedder <m>]` | Pillar 3 sink-side matcher for this run (FR-006); overrides `security.egressMatcher`. |
| `run <spec> --no-mcp-quarantine` | Register even the tools of servers `mcp doctor` marked chronically failing (default: withdraw them + inject a notice). |
| `dev <spec> [--once] [--debounce <ms>] [--plugins <a,b>]` | The authoring dev-loop: compile the spec in memory, run the emitted bundle as a **supervised child** (`CREWHAUS_TRACE=pretty` by default — turns stream live), and recompile + relaunch it on every spec / `.crewhaus/commands` / skills change. A broken edit keeps the running child; a crashed daemon shape restarts (bounded); each completed turn prints a `[dev]` summary. `--once` launches one run to completion and exits with its code (a credential-free CI boot check); `--debounce` sets the change-coalescing window in ms (default 150). |
| `serve --mcp <spec> [--sse] [--port <n>] [--plugins <a,b>]` | Project the spec's **cli** agent as an **MCP server** so it becomes a tool inside Claude Code / an IDE / another CrewHaus runtime. `--mcp` is the only projection kind today. Default transport is stdio; `--sse` serves over HTTP (a `fetch` server, default port 8000 or `CREWHAUS_MCP_PORT`), overriding the spec's `expose.mcp.transport`. A **channel** spec does not use this verb — it self-exposes from its own compiled daemon; see [`expose.mcp`](#exposemcp--a-bundle-as-an-mcp-server). |
| `runs resume <session> [--spec <path>] [--prompt <text>]` | Re-drive a persisted cli session — e.g. one PARKED on a pending approval (resolved via `approvals grant`). Resolves the backing spec from `--spec` or `cwd/crewhaus.yaml`; the run flags a resumed continuation honours (`--model`, `--budget-usd`, `--trace`, `--streaming`, `--justification-judge`, `--egress-matcher`, `--user`, …) are threaded verbatim. `runs` is the run-lifecycle namespace; `resume` is its only CLI verb today. |
| `approvals list\|show\|grant\|deny <id> [--dir <root>] [--by <who>] [--json]` | Resolve tool-permission approvals a headless run parked under `permissions.ask_mode: pause`: `list`/`show` the pending queue, `grant`/`deny <id>` a decision (recorded with the deciding identity from `--by` / `CREWHAUS_USER`, default `cli`). `grant --once` applies to the next matching tool call only (the default; the runtime consumes a grant on use). After granting, resume the parked run with `runs resume <session>`. A grant is bound to the input the operator SAW: if the resumed run's model regenerates the call with different wording, the runtime executes **the approved input**, not the regenerated one, consumes the grant, and tells the model it did so. That is what makes a one-shot grant able to complete a model-driven tool call at all — before 0.5.4 any re-worded argument re-parked the run under a new id, and only `--always` (a standing allow) got a call through. |
| `export claude-plugin <spec> [--out <dir>] [--force] [--author <name>] [--author-email <e>] [--description <d>]` | Emit an Anthropic-compatible **Claude Code plugin** directory from any target shape. Output defaults to `<cwd>/<pluginName>` and refuses to overwrite a non-empty dir without `--force`. `--author` (default `CrewHaus`) is stamped into `.claude-plugin/plugin.json` (Anthropic's schema requires a non-empty author). |
| `upgrade [spec] [--write]` | Detect the spec's `version:` drift vs the current CLI and run the migration chain (validated). Dry-run diff by default; `--write` applies. |
| `migrate-all --from <N> --to <N> [--dry-run]` | Batch-migrate every spec in the registry to a newer IR version. |
| `context --bundle [-o <file>]` | Emit a single-markdown orientation manifest (`--factory-root` / `--docs-root` / `--demos-root` locate the sources). |

`init` scaffold flags (`--ci`, `--with-evals`, `--sentinel`, `--suite`) are
composable with each other and with an existing harness; `--force`
overwrites a previously scaffolded workflow or eval assets (never the spec).

### `expose.mcp` — a bundle as an MCP server

Two ways a CrewHaus agent becomes a tool for something else, and they are not
interchangeable:

| Shape | How | Transport |
|---|---|---|
| `cli` | `crewhaus serve --mcp <spec>` — the CLI hosts it | `stdio` (default) or `--sse` |
| `channel` | the compiled daemon self-exposes | `sse`, on its own public port |
| `managed` | **not wired** — compile warns | — |

A channel spec opts in and recompiles:

```yaml
expose:
  mcp:
    transport: sse       # `stdio` mounts nothing on a daemon — that is `serve --mcp`
```

The daemon then serves MCP at **`/mcp`** on its public port (`PORT`, default
3000) and prints the URL at boot. The projected `chat` tool runs one turn
through the same loop a channel message drives — same memory fabric, same
boundary classifier, same permissions.

**It is authenticated**, unlike the rest of that port. The adapter routes are
signature-verified per platform and `/healthz` is deliberately public, but MCP
drives whole agent turns with the bundle's tools and credentials. The bearer is
`CREWHAUS_MCP_TOKEN`, or a 32-byte token minted `0600` into
`.crewhaus/run/mcp-token` at every boot (a token left by a dead daemon must not
authenticate against its replacement). Boot prints which is in use.

```
Authorization: Bearer <token>
```

Each MCP session gets its own harness session, so two IDEs driving one daemon
do not share a transcript.

`tools: per-subagent` projects as `chat` on channel and compile says so: the
channel turn function takes no routing argument, so per-sub-agent tools would
all drive the same undirected turn. Sub-agents still run *inside* the turn.

**Consuming one.** `mcp_servers: { peer: { transport: sse, url: … } }` reaches
either HTTP MCP revision — the client tries Streamable HTTP (the 2025-03-26
revision, which our own `expose.mcp` serves) and falls back to the legacy
HTTP+SSE transport, remembering which one worked. Before 0.5.4 it spoke only
the legacy transport, so a CrewHaus peer could never consume a CrewHaus
`expose.mcp` endpoint — the symptom was `SSE error: Invalid content type`.

### Optional MCP peers — `required: false`

```yaml
mcp_servers:
  peer:
    transport: sse
    url: https://peer.internal/mcp
    required: false   # default: required — a failed boot stops the run
```

By default a server that cannot connect at boot fails the run: an agent whose
instructions assume a tool behaves worse when it silently vanishes than when
it refuses to start. `required: false` flips that for peers whose absence is a
normal state — the classic case is two channel daemons that mount each other
over `expose.mcp`, which otherwise cannot both start first and end up
`crash-looping` under a supervisor.

An optional server that fails at boot — unreachable, or an `$ENV` secret
unset — warns and the run continues without its tools. What happens next
depends on the shape:

- **Channel daemons, batch workers, research runs** keep reconnecting in the
  background and register the peer's tools when it arrives. These shapes
  re-read their tool catalog per message / job / branch, so the tools reach
  the model without a restart. Boot does not wait for the attempt — an
  unreachable peer answers slowly, and blocking startup on it is the failure
  this setting exists to remove.
- **cli, crew, workflow, and eval runs** are one-shot: the tool list is frozen
  at boot, so there is no retry — the tools are simply absent for that run,
  and the connection is torn down rather than left reconnecting behind a
  finished run.

The Hangar's pre-spawn preflight (the gate behind `crewhaus hangar`, and
`GET /api/h/:id/preflight`) names each optional server, and an unset secret on
one is reported as a warning instead of a blocker — it degrades the peer, it
does not stop the harness.

### `ListTools` — what tools does the agent have right now?

Every tool-carrying run also advertises a built-in, read-only `ListTools`
tool. Calling it lists the tools bound to the **current** request — name,
flags (read-only / destructive / gated), one-line description. That is
runtime truth: any enumeration earlier in the transcript can be stale,
because tools arrive mid-session (an optional MCP peer connecting, a skill
activating) and vanish (`mcp doctor` quarantine). `ListTools` is auto-allowed
in every permission mode, so a headless run never parks on it.

The runtime also notices for the model: when a resumed session's toolset
differs from what the previous turn advertised, a one-line system marker in
the transcript names what was added and removed — and suggests calling
`ListTools` to verify.

### `thredz.messaging` — agent-to-agent messaging

`thredz:` is the MEMORY knob and stays that way: turning it on aliases the 18
wiki/goal/space tools and nothing outward. The nine A2A messaging tools
(`agent_register`, `agent_update`, `agent_list`, `message_send`, `inbox_poll`,
`message_ack`, `thread_get`, `agent_block`, `agent_unblock`) register only when
the spec asks:

```yaml
thredz:
  api_key: $THREDZ_API_KEY
  messaging: true      # the nine A2A tools — default off
  agents: true         # register this agent's handle at boot, named from the spec
```

Reads are read-only; directory and mailbox mutations are destructive; and
`message_send` alone carries the justification gate — it puts text in front of
another agent, which is the visible-side-effect class the gate exists for.
Against a pre-0.3.0 thredz-mcp the messaging tools simply come back as
unadvertised and the boot warns, never fails.

---

## Health and diagnostics — `doctor`

One command, many probes. Bare `doctor` checks environment health (Bun
version, credentials, a working spec, Docker for sandboxed tools). The
flags below are additive report/gate modes.

| Command | Purpose |
| --- | --- |
| `doctor` | Environment health check. |
| `doctor --detect [--no-probe]` | Read-only inventory of reachable providers, the local Ollama/vLLM endpoint's models, and MCP servers imported from `.mcp.json` / Claude Desktop config. `--no-probe` skips the localhost HTTP probe (offline / CI). |
| `doctor --fix` | Apply the mechanical remediations `--detect` otherwise only prints (scaffold `crewhaus.yaml`, create `.crewhaus/`, mark outward tools `scope: external`, append commented `.env` stubs). **Dry-run is the default** — without `--fix`, doctor prints the diff it would apply. |
| `doctor --philosophy-alignment` | The three-pillar architectural audit. |
| `doctor --philosophy-alignment --json [--baseline \| --accept-baseline]` | Scope-audit drift gate: `--json` persists findings to `.crewhaus/scope-audit/<date>.json`; `--baseline` exits non-zero only on **new** findings vs the pinned baseline; `--accept-baseline` promotes the current findings to the baseline. |
| `doctor --context-pressure [--sessions <N>]` | Report over recent sessions: truncation recoveries, compaction fires, snip-vs-autocompact, plus the spec knobs and `advise`/`optimize` hints to relieve it. Report, not gate — always exits 0. |
| `doctor --models` | Model advisory: flags spec models missing from the pricing table (silently billed $0), pricing-table staleness, and known provider sunsets. |
| `doctor --slo` / `--ttft` | Compare recent p95 TTFT against the spec's `observability.slo.ttft_ms` and name faster candidates on a breach. Container-HEALTHCHECK exit semantics (0 within/no-data, 1 breach). |
| `doctor --liveness` | Process-liveness only — exit 0 fast, no credential/spec checks. The probe target for container HEALTHCHECKs and k8s exec probes. |

---

## The self-building eval flywheel

The nightly loop that lets a harness build its own evals, learn from real
usage, and improve itself — always accept-then-write, always regression-gated.

`crewhaus eval` runs a `target: cli` spec and refuses any other shape. To
evaluate one of the other target shapes, emit its bridge with
`compile --with-eval-harness` (see
[Build, run, and author](#build-run-and-author)) and run that bundle.

### Running an eval

| Command | Purpose |
| --- | --- |
| `eval <spec> --dataset <d> --graders <g>` | Run the agent against a dataset and grade it (deterministic graders + LLM-as-judge). `--dataset` also accepts `registry:<ref>` (bare = train+dev). Flags: `--judge-model`, `--concurrency`, `--seed`, `--repeats`, `-o`. |
| `eval … --repeats <K>` | Run every sample K times (trial *t* gets `seed+t-1` when `--seed` is set; i.i.d. draws otherwise). Trial 1 is the canonical result; per-trial grades land on the sample (`results.json` `trials[]`) and the aggregates gain **pass@K** (at least one trial passed — the optimistic capability metric) and **pass^K** (all K passed — tau-bench's reliability metric: a flaky 60%-reliable agent scores 0.6^K). Trials run sequentially inside each sample's concurrency slot, so a K-repeat run costs ~K× the wall clock and spend (`tokens_all_trials` makes the real spend visible). Samples whose trials disagreed are additionally flagged `flaky` (counted + listed in `aggregates.flaky` / `flakySampleIds`, and marked on the run-history entry) — **verdicts are untouched**; quarantining a flaky sample stays a human decision. |
| `eval … --slice <k1,k2,…>` | Group results by string `metadata` values (default `family,difficulty,language,source`, applied only where present). Per-slice `{sampleCount, passRate, meanScore}` land in `results.json` `slices`, one `[eval] slice <key>:` line per key on stdout, and a sortable table in `index.html`. Computed by the runner, so `--models` cells and `target: eval` bundles inherit it; a metadata-less dataset produces byte-identical output. |
| `eval … --gate` | Exit non-zero on a regression vs the pinned `(spec, dataset)` baseline (strict: any pass-rate drop or pass→fail flip). Samples a judge abstained on in *either* run leave the flip comparison, and the run says so (`[eval] gate: excluding N abstained sample(s)…`). A **partial** (budget-exhausted) run can never pass — an incomplete measurement cannot honestly clear a pre-declared gate. Baseline pins now also record `gradersHash` and (when `--judge-model` pinned one) `judgeModel`: on a mismatch the run warns loudly and starts a **new baseline lineage**, exactly like the dataset-changed path. |
| `eval … --max-p95-latency-ms <N>` | Pre-declared ops criterion joined to the baseline gate: fail the verdict when p95 per-sample latency rose more than N ms **vs the pinned baseline**. The first run pins and is not gated. Rejected alongside `--sentinel` (which has its own drift gate) and `--models` (whose cells skip the baseline lineage). |
| `eval … --max-cost-usd <F>` | The second ops criterion, and an **absolute** ceiling rather than a baseline comparison: fail when the run's estimated cost exceeds `$F`. The figure is the **total** — agent *plus* judge/grader spend — through the same pricing seam that prints the `[eval] cost:` line, so a judge-heavy run cannot slip past a ceiling by metering only its agent half. A pricing miss on *any* model in the run leaves the total unknown, and the gate then **warns instead of failing**. Rejected alongside `--sentinel` / `--models`. |
| `eval … --sample-timeout-ms <N>` | Per-sample agent-invocation wall-clock watchdog (default: the spec's `limits.deadline_ms`). A timed-out sample records an errored result with full artifacts instead of stalling a concurrency slot. |
| `eval … --budget-usd <F>` | Run-level agent-model spend cap (default: the spec's `budget.usd`). At the cap, in-flight samples finish and queued samples abort with `[eval] budget exhausted after k/N samples`; `results.json` is marked **partial** (a partial run never pins a baseline). Eval always **stops** at the cap — the spec block's `on_exceed: degrade` ladder does not apply here. Judge/grader spend is not metered against it; an unpriced model disables enforcement with a loud warning. Under `--models`, each cell meters its own cap. |
| `eval … --record-tools <dir>` | Append every tool execution's result to `<dir>/tools.jsonl`, keyed by `(sampleId, toolName, sha256(canonical-JSON args))`. Tools still run for real and the run is otherwise byte-identical. A recording holds tool args and results **verbatim** — treat `<dir>` like a session transcript. |
| `eval … --replay-tools <dir> [--replay-miss error\|live]` | Serve those recorded results instead of executing. The interception point is `RegisteredTool.execute`, so built-ins, MCP tools, Skill/Task wrappers and memory-fabric tools are all covered. `--replay-miss` defaults to **`error`** (fail that sample naming the missing key, never noise-retried); `live` executes for real. Repeated identical calls replay in order and then keep replaying the last one, printing an `[eval] warning:` naming the reused calls plus a `reusedEntries` count. **Scope is tools only** — MCP servers still boot and the model still runs live, so a replay is neither offline nor credential-free. |
| `eval … --resume <runDir>` | Re-open an interrupted run under its **original** `runId` + `startedAt`: every sample that already wrote `grades.json` is reloaded (no agent or judge call, no spend), only the missing ones run, and the union is re-aggregated into a fresh `results.json` + `index.html`. Refuses loudly, before any spend, when the run's `specHash` / `datasetHash` / `gradersHash` no longer match `run.json`. An errored sample counts as complete (delete its artifact dir to re-run just that one); under `--repeats` a sample re-runs whole unless every trial directory is complete. `--budget-usd` is **re-armed per attempt**, so the resume prints `run.json`'s cumulative `spentUsd` before it spends anything. The resumed run appends a *superseding* `index.jsonl` entry under the same runId (the index stays append-only). When the pinned baseline *is* the run being resumed, the gate is refused with a warning rather than comparing the run to itself. Mutually exclusive with `-o` and `--models`. |
| `eval … --allow-test-split` | Consume an explicit `#test` registry ref — the locked holdout, meant for release-gate runs. Bare refs stay train+dev regardless. |
| `eval … --no-preflight` | Skip the pre-spend lint-lite. By default, before any model call, duplicate sample ids and an all-gold-less dataset paired with gold-needing graders **refuse the run**; partial gold gaps warn on stderr and proceed. |
| `eval … --no-promote` | Keep the existing baseline pin instead of auto-promoting this run on a gate pass. |
| `eval … --models <m1,m2,…>` | Benchmark matrix: run the same dataset+graders once per model; each cell writes to `<out>/<model-slug>/` and the run emits `matrix.json` + `index.html`. Incompatible with `--gate`/`--no-promote`, with `--max-p95-latency-ms`/`--max-cost-usd`, with `--record-tools`/`--replay-tools` (cells share sample ids), and with `--resume`. |
| `eval … --no-regressions` | Skip the default union of the per-spec `<name>-regressions` registry dataset (also skips the failure-arbiter's bug-sample pin into it). |
| `eval … --no-retry` | Opt out of the runner's default one-shot retry of ERRORED samples (infra noise, not graded failures). |
| `eval … --sentinel --baseline <run-dir>` | Nightly model-drift sentinel: re-run the seed-pinned dataset against the unchanged spec and diff against a frozen baseline; a flip/score-shift when specHash **and** dataset-hash are both unchanged is provider drift → exit non-zero. |
| `eval … --voice [--replay-dir <d>] [--max-ttft-ms N] [--max-turn-latency-ms N] [--max-barge-in-yield-ms N] [--graders <g>]` | Voice replay eval: replay recorded call-session logs through the voice grader pack (latency / barge-in / transcript). With `--graders`, each replayed transcript is also **content-graded** by the ordinary grader stack, and a content failure fails the session exactly like a latency breach; a replay carries no gold, so gold-needing graders (`exact_match` / `expected_contains`) say so rather than guessing. Without `--graders` the path is byte-identical and still credential-free. `--voice` does not support `--record-tools` / `--replay-tools` / `--replay-miss` / `--resume`. |
| `eval coverage [--sessions N\|all] [--dataset <d>] [--graders <g>] [-o <dir>] [--format text\|html\|json]` | Detect production behaviors (tool / MCP / bigram / compaction patterns) that no eval sample exercises, ranked by production frequency. The `json` form is a backlog for `dataset mine`. `--graders` adds the grader-side join — how many samples each grader can actually score (sharing `dataset lint`'s own gold predicate, so the two surfaces cannot disagree), which declared graders no recent run ever recorded, and which judge criteria never varied across recent runs (a dead criterion pays judge tokens and can never change a verdict). Omitting it leaves every rendered byte unchanged; bare registry refs are inspected across **all** splits so gap analysis is not run over a partial record. |

Every run additionally reports what the numbers can and cannot support.
Abstained samples (a judge that declined to score) leave the pass-rate
denominator and `meanScore` and land in `needsHuman` / `needsHumanSampleIds`
(`[eval] needs_human=N: <ids>`); samples where a judge **panel** split with
high vote entropy still **count** but land in the separate `needsReview`
bucket; `metadata.source: "canary"` samples are excluded from pass rate and
`meanScore` and reported as their own `canary` count. The three buckets are
disjoint. `aggregates` also carry `passRateCI95` (Wilson) and
`meanScoreCI95` (Student *t*) — printed on the `[eval]` summary line and
**absent** where the data cannot support them rather than fabricated — and
`judgeUsage`, so the `cost:` line breaks out agent vs judge vs total (an
unpriced model renders `n/a`, never a fabricated `$0.0000`). The two cost
knobs deliberately meter different things: `--budget-usd` bounds **agent**
spend — the quantity a spec's `budget.usd` declares, so wiring a judge into
a rubric can never silently shrink an existing run's sample budget — while
the `--max-cost-usd` gate ceiling is checked against the **total**.

**Spec blocks now honoured (behaviour change).** `limits:` and `budget:`
were silently dead inside `crewhaus eval`. `limits.deadline_ms` now bounds
each sample's invocation, and the remaining ceilings (`turn_timeout_ms`,
`model_call_timeout_ms`, `max_tool_iterations`, `max_concurrent_tools`,
`context_limit`, `loop_detection`) thread into each sample's chat loop
exactly as `crewhaus run` threads them. A flag always beats the spec.

### Suites, planning, and red-teaming

| Command | Purpose |
| --- | --- |
| `eval suite <suite.yaml> [--tier fast\|nightly\|release] [--spec <s>] [-o <dir>] [--gate]` | Run one named CI **tier** of a suite manifest. `--tier` defaults to `fast`; `-o` defaults to `.crewhaus/evals/suite_<tier>_<timestamp>`; `--spec` overrides every entry's spec (what the CI scaffold points at the base checkout); `--gate` maps a failing tier to a non-zero exit. Entries run **sequentially** through the same code path a hand-typed `crewhaus eval` takes — registry refs, regression union, preflight, triage, run history and baselines all behave identically — into `<out>/<entry>/`, and the tier passes only when **every** entry passes. A preflight refuses missing spec/dataset/graders files before the first entry spends anything; a crashed entry is isolated so the rest of the tier still reports; a budget-**partial** entry always fails. The verdict plus each entry's aggregates and failure reasons land in `<out>/suite.json`. |
| `eval plan --target-delta <F> [--confidence <C>] [--pilot <runDir>]` | Offline sample-size planner: `n ≈ z²·p(1−p)/e²`, printed with every term and where it came from (which *z* for the confidence, which *p* — a pilot run's measured pass rate, or the variance-maximizing 0.5 worst case — and which *e*), the substituted arithmetic, and the doubled budget a two-run comparison needs. With `--pilot` it also reports the smallest delta that pilot's own *n* could ever have resolved. `--confidence` defaults to **0.95**; an unlisted confidence snaps to the nearest tabulated *z*. No model call, no credentials, no spend. |
| `redteam generate [--spec <s>] [--taxonomy <t>] [--count N] [--seed N] [--budget-usd F] [--model <m>] [--out-dataset <n>] [--out-graders <f>] [--force]` | Generate an attack suite against the agent — behaviour categories × attack strategies, deterministic and offline (attack strings are composed from inert parts, never shipped whole), with optional model-rephrased variants under the budget. Emits a `<spec>-redteam` registry dataset (synthetic/adversarial, and **never gated by default** — consume it with an explicit `--dataset registry:<spec>-redteam`) plus a paired refusal-grading `graders.yaml`. `--count` defaults to 24. |
| `redteam report --runs <dir\|dir,dir\|last:N>` | Attack-success rate by category and strategy over persisted runs. Errored and judge-abstained probes **leave the denominator** — they are never counted as resistance. Samples carrying no red-team provenance are reported and ignored. |

The suite manifest's tier vocabulary is fixed (`fast` / `nightly` /
`release`) and both the manifest and each entry are strict, so a typo'd key
or an invented tier name is refused at parse. Each entry declares
`{name, dataset, graders}` plus optional `spec`, `seed`, `repeats`,
`concurrency`, `slice` (a list), `gate`, `allow_test_split`, and a strict
`thresholds` block (`min_pass_rate`, `min_mean_score`,
`max_p95_latency_ms`, `max_cost_usd`). The two ops thresholds are criteria
**of** the baseline gate, so declaring one without `gate: true` is refused
as dead config — and so is an entry declaring **neither** thresholds nor
`gate: true`, because it could never fail. Absolute floors are read from
each entry's own `results.json` and bite from run one, including in a fresh
CI workspace; `gate: true` is the unchanged baseline gate, vacuously passing
until something pins.

### Reporting and analysis

| Command | Purpose |
| --- | --- |
| `eval-report diff <prevRun> <newRun> [-o <dir>] [--seed N] [--epsilon F] [--pairwise [--judge-model <m>]]` | Compare two eval runs and emit a diff report highlighting pass/fail flips, now with **paired significance testing** always on: a sign-flip permutation test over paired per-sample pass-rate deltas on shared ids (abstained-on-either-side pairs excluded, the same exclusion the gate applies), exact enumeration at paired *n* ≤ 20 and a seeded Monte Carlo above it. Output adds the pass-rate delta with a bootstrap 95% CI, a two-sided p-value, the paired *n*, and a plain-language significant/not-significant verdict — to stdout, to `significance` in `diff.json`, and to the HTML header. **The strict gate never consults significance.** `--seed` pins the draw (unseeded diffs of the same runs are already byte-identical); `--epsilon` is the score-shift tolerance, **default 0.1** — flips are never subject to it. Per-slice deltas for the `(key, value)` pairs both runs share land in `sliceDeltas` + a table in the HTML and stdout. |
| `eval-report diff … --pairwise` | Opt-in head-to-head judging: for every shared sample the judge compares the two runs' outputs **twice** with presentation order swapped (fresh injection sentinels, a forced `submit_comparison` tool, temperature pinned 0). Reports the new side's win rate with ties counted half, plus an order-consistency figure — a verdict that flips with the order is position bias by construction and consolidates to a **tie**, never a win. Costs **2 judge calls per shared sample** and dies with a clear message without visible judge credentials. |
| `eval-report history [--spec <n>] [--dataset <n>]` | List recorded runs from `.crewhaus/evals/index.jsonl`. Runs containing flaky samples are marked. |
| `eval-report trends [--spec <n>] [--dataset <n>] [-o <dir>]` | Pass rate / mean score / cost **over time** per `(spec, dataset)`, folded from the same index: a per-run table plus a movement line per lineage (first → last, delta in percentage **points**). `-o` writes a self-contained `index.html` (inline CSS + a hand-built inline SVG chart, zero external assets) and `trends.json`. Fully offline — no run directory is opened. |
| `eval-report export --runs <dir\|dir,dir\|last:N> --format csv\|jsonl [-o <file>]` | Flatten runs to one row per **(run, sample, grader)**: run config (`runId`, `ts`, `specHash`, `dataset`, `model`, `judgeModel`, `seed`), the sample's verdict, latency, trial pass rate, flaky flag and slice membership, then each grader's `passed` / `score` / `abstained` / `rationale` (clipped, newline-flattened). A sample whose graders never ran **still emits a row** — dropping errors is how a pass rate lies. A moved or unreadable run directory is reported on stderr and skipped, never silently omitted. |
| `eval-report baseline show [--spec <n>] [--dataset <n>]` | Print pinned baselines from `.crewhaus/evals/baselines.json`. |
| `eval-report baseline set <runId>` | Pin a recorded run as its `(spec, dataset)` baseline (carrying `gradersHash` / `judgeModel` onto the manual pin). |
| `eval history\|baseline\|baselines\|diff …` | Working **aliases** for the `eval-report` verbs above: a one-line stderr notice names the canonical verb, then flags and positionals pass through verbatim. Only these exact bare words alias — a spec file literally named `history.yaml` still takes the run path. `trends` and `export` are **not** aliased; spell them `eval-report trends` / `eval-report export`. |

### Optimize, flywheel, and scheduling

| Command | Purpose |
| --- | --- |
| `optimize <spec> --dataset <d> --graders <g>` | Active eval-driven optimization. `--dataset` accepts `registry:<ref>` (bare = train+dev; `#test` is refused outright). Flags: `--mutator rule-based\|claude\|meta-harness`, `--iterations`, `--seed`, `--concurrency`, `--improvement-threshold`, `--budget-usd` (stop a model-driven run before it exceeds $N), `--write-back`, `-o`. |
| `optimize … --stage <name>` | Multi-stage specs: `optimize` now accepts **workflow / graph / crew / pipeline** specs, not just `cli`. `--stage` narrows the search to one workflow step / graph node / crew role — an unknown name errors and lists the valid ones. **Without** it, every stage optimizes sequentially in declaration order, each gated independently: a winning stage composes into the working spec the next stage starts from, a losing stage leaves the spec untouched. `--iterations` is **per stage**; `--budget-usd` stays a **run** ceiling, threaded down as remaining budget so a three-stage run cannot spend 3× the cap. Only per-stage prompt paths already in `OPTIMIZABLE_PATHS` are rewritten, and `kind: judge` steps/nodes run no agent turn so they are never mutated. Refused alongside `--from-advice`. Each candidate is compiled with the same eval-entry emission `compile --with-eval-harness` performs, so run it inside a harness whose dependencies are installed (the default `-o` already is). |
| `optimize … --mutator meta-harness` | **Experimental** third mutator (every run prints a notice). Same model-backed proposer as `--mutator claude` and the same accept gate, `--budget-usd` meter and `OPTIMIZABLE_PATHS` validation — what differs is the proposer's input: it reads the run's filesystem-backed **experience store** (every prior candidate's artifact, per-sample scores and trace) instead of a summary window. Deliberately spec-shaped: the CLI proposer returns replacement *instructions* that still round-trip through `parseSpec`; bundle rewriting stays library-only. |
| `optimize … --ratings <session>\|all [--no-redact]` | Inline-distill user ratings into the training set (synthesizes the dataset, and the graders when `--graders` is omitted). Sample text is PII/secret-redacted by default — the same detector set as `crewhaus distill` — which keeps raw text out of the sample pool, the synthesized graders, and the meta-prompt sent to the mutator model. `--no-redact` opts out (dev/local only). |
| `optimize … --from-advice <suggestions.json>` | Apply `advise` SpecPatches through the eval-gated accept/reject/compose loop instead of running the mutation search. |
| `optimize … --few-shot <pool>\|auto [--few-shot-k N]` | Inject the top-K harvested few-shot examples at the front of the seed instructions the optimizer mutates. Injection runs **after** the dataset is materialized and drops every pool example whose `(sessionId, turnNumber)` provenance appears in the eval dataset's own `metadata.sessionId` / `metadata.turnNumber` stamps, printing `[optimize] few-shot: excluded N pool turn(s) overlapping the eval dataset` — counted, logged, never silent. A dataset with no provenance metadata excludes nothing; if **every** pool example overlaps, the run refuses rather than injecting nothing. |
| `optimize … --no-pin-regressions` | Skip pinning an accepted patch's fail→pass recoveries into `<name>-regressions`. |
| `flywheel run [spec]` | The whole loop, one command: compile gate → baseline eval → optimize → after eval → acceptance gate (pass rate strictly up **and** zero regressions) → write-back on accept. A rejected patch never touches the spec. Flags: `--dataset`, `--graders`, `--budget-usd`, `--iterations`, `--seed`, `--concurrency`, `--mutator rule-based\|claude`, `--dry-run` (rehearse, never write), `--allow-dirty` (opt out of the clean-tree invariant). Every run now **prints the dataset it resolved and why** — `[flywheel] dataset: <resolved> (source: flag\|convention\|ratings-registry)` — and when a conventional `eval/dataset.jsonl` shadows an existing `<spec>-ratings` registry dataset it warns with the exact remediation (`pass --dataset registry:<spec>-ratings…`). |
| `flywheel run … --gate-split train\|dev` | Narrow the before/after **acceptance** evals to one registry split (the optimizer's own train/dev sets are unchanged). A split-gated run keys into its own baseline lineage `<name>@<version>#<split>`. Refused for flat-file datasets (no split boundaries) and for `#test`. Omitted, behaviour is exactly as before. |
| `flywheel init [--force] [--suite <suite.yaml>]` | Scaffold `.github/workflows/crewhaus-flywheel.yml` (nightly cron + manual dispatch). Accepted improvements arrive as PRs for human review — never auto-merged. `--suite` appends a nightly-tier `eval suite` step to the same cron, after the improvement PR is opened. |
| `schedule generate --for flywheel\|eval-gate\|sentinel [--runner cron\|launchd\|systemd] [--dir <path>] [--spec <s>] [--dataset <d>] [--graders <g>] [--baseline <b>]` | Print ready-to-install scheduling text wrapping the matching `crewhaus` command, for teams not on GitHub Actions. `--runner` defaults to `cron`. A shim by design: **it prints, it never installs.** |
| `scaffold-evals <spec> [-o <dir>] [--samples N] [--model <m>] [--force]` | Day-one eval assets **from the spec**: sample stubs derived from `agent.instructions` (one model call with credentials, deterministic template without) + one starter grader (a spec-goal `llm_judge` rubric online, a non-empty-answer floor grader offline). |
| `scaffold-evals <spec> --template rag\|summarize\|extract\|support\|safety\|classify` | Start from the first-party eval-template library instead of drafting from the spec: the family's fully-anchored `graders.yaml` is copied verbatim under a provenance header, and `dataset.jsonl` is seeded from the family's samples and topped up to `--samples N` with spec-derived stubs. `classify` is the exception — every grader there needs a gold, so its dataset stops at the gold-carrying seeds and says why. **Offline by construction**: the families are embedded static content, so `--model` is refused with `--template` and nothing is fetched. An unknown family lists the available ones instead of guessing. |

`crewhaus init --ci --suite <suite.yaml>` scaffolds the **tiered** CI
workflow instead of the single-run one: the fast tier on every PR (a
base-branch spec run pins baselines in the fresh workspace, then the PR spec
runs with `--gate`), a nightly-cron job for the nightly tier, and a
tier-verdict PR comment built from `suite.json`. `crewhaus init --sentinel
--suite` gives the drift cron a nightly-tier step that runs **even when the
probe failed**. In all three scaffolds the suite path must be
harness-relative and inside the harness; a not-yet-written manifest warns
rather than failing, and an existing one is parsed so a tier the YAML runs
but the manifest never declares is named at scaffold time. Without
`--suite`, every scaffold is byte-identical to before.

---

## Datasets, graders, and judges

The versioned dataset registry, grader drafting, dataset growth from
production, and judge calibration.

### The dataset registry

| Command | Purpose |
| --- | --- |
| `datasets list` | All registered datasets + versions. |
| `datasets get <name>[@version] [--split train\|dev\|test]` | Print a dataset's samples as JSONL (one split, or the all-splits merge). Still prints test rows — inspection is not consumption — but says so on stderr. |
| `datasets put <name> --file <f.jsonl> [--split-spec 70/15/15 \| --split <name>] [--canary]` | Import a file as a new auto-bumped version. `--canary` injects exactly **one** contamination-tripwire sample whose input is a deterministic 32-hex phrase derived from the `(name, version)` hash — no wall clock, so the same version always produces the same canary. It is tagged `metadata.source: "canary"` and carries **no gold**, and every eval excludes it from the pass-rate denominator. |
| `datasets verify <name>[@version]` | Recompute every split's per-sample content hashes and compare them to what the record stored at `put`. Omit the version to check every version. Offline, and **exits non-zero on any mismatch** — the registry-integrity CI gate. |
| `datasets status <name> [--runs N]` | Freshness and saturation. Joins registry versions with `.crewhaus/evals/index.jsonl` (`--runs N`, default **10**) and reports per-version age from `createdAt`, indexed-run count per version and when it last ran, how many runs consumed the locked `#test` split, and the test-split burn count. Sample ids that appeared in ≥ 2 of the last N joined runs and passed **every** time are listed as rotation candidates — the saturation signal. |
| `datasets release <name>[@version] --spec <s.yaml> --graders <g.yaml> [--force]` | The sanctioned holdout spend: run `crewhaus eval` over the version's locked `#test` split (threading `--allow-test-split`, and skipping the regression union so the holdout stays pure), then append a release entry `{version, runId, ts, passRate}` to the record. A version whose test split was already released **refuses a second release without `--force`**, which warns that a re-run holdout score is no longer a first look. |
| `datasets card <name>[@version] [-o <file.md>]` | Markdown datasheet for a dataset version: split sizes + sample-hash counts, the all-splits content hash, `createdAt` and age, a provenance breakdown by `metadata.source` (percentages, untagged counted), the indexed eval-run count, the full release/burn history, and an embedded offline lint summary. Never mutates the record; stdout by default. |

### Growing and auditing a dataset

| Command | Purpose |
| --- | --- |
| `dataset mine [--sessions N\|all] [--out-dataset <name>] [--review [--yes]] [--no-redact]` | Mine hard cases from session struggle signals into a quarantine dataset; `--review` promotes accepted candidates into a mined registry dataset (interactive in a TTY; `--yes` required to accept non-interactively). The signal set is now `tool-error \| error \| loop \| retry \| egress-block \| eval-fail`; the new **`eval-fail`** signal is read from each session's trace sidecar `<id>.events.jsonl`, and a turn the `on_fail: retry` ladder *recovered* is deliberately **not** harvested — one that burned the ladder and still failed is flagged `eval_retries_exhausted`, with the judge score / threshold / grader riding into the quarantine sample's metadata. Candidate text is PII/secret-redacted by default; `--no-redact` opts out. Mined candidates also enqueue **pointers** into the review queue (the quarantine JSONL stays the payload store). |
| `dataset synthesize --from <file\|registry:ref> [--count N] [--budget-usd N] [--out-dataset <name>] [--model <m>]` | PII-redacted stress variants (paraphrase / truncate / ambiguate / inject) into a separate synthetic split that never contaminates human golds. Variants are now stamped `metadata.source: "synthetic"` (previously the tool-named `"synthesize"`), and paraphrase variants — template *and* model — additionally carry `metadata.paraphrase_group` (the parent sample's id) so the `consistency.paraphraseGroup` pack can score them; truncate/ambiguate/inject deliberately change the question and stay group-less. A bare registry `--from` resolves train+dev and `#test` is refused. |
| `dataset refresh-goldens --dataset <file\|registry:ref> [--min-score F] [--apply] [--model <m>]` | Reconcile user corrections and up-rated turns with existing golds; propose gold updates as a review diff. `--apply` writes a **new** registry version — never in-place — patching golds **within** the record's existing split structure, so unselected splits (test included) pass through byte-identically. When a human-evidence proposal golds a `synthetic` sample, `--apply` retags it `synthetic_human_verified`; the registry refuses a `synthetic` sample carrying an `expected_output` outright. |
| `dataset audit [--pii] --dataset <file\|registry:ref> [--apply] [--strict]` | Offline PII/secret scan of an existing dataset — regex detectors only (the shared set `dataset synthesize` and `fewshot harvest` redact with), no model calls. The report counts hits per detector, field and sample id and **never echoes the matched text**. A registry ref without `#split` is scanned across **all** splits, test included (inspection, not consumption). Multi-turn aware: history message contents scan as `history[<i>].content`. `--apply` requires a `registry:<name>[@version]` ref and writes the redacted samples as a **new** auto-bumped version preserving the split structure exactly (never in place, never re-split, every turn kept with roles verbatim). `--strict` exits non-zero on any hit. |
| `dataset lint (--dataset <file\|registry:ref> \| --all) [--graders <g.yaml>] [--strict]` | Offline hygiene lint, no model calls. **Errors:** duplicate sample ids; empty-string golds; gold-needing graders (`exact_match` / `expected_contains`) declared when **no** sample carries a gold; a `--canary` phrase found in `crewhaus.yaml` or a `.crewhaus/fewshot` pool (contamination). **Warnings:** ids reused with *different* content in other versions of the same registry dataset (same-content reuse is normal lineage); near-duplicate inputs (normalized token overlap ≥ 0.9); `expected_tools` on samples when the conventional spec exposes no tools; a `metadata.source` outside the provenance taxonomy, with the offenders listed. Graders are found via `--graders`, else the conventional `eval/graders.yaml`. Registry refs lint **every** split; `--all` sweeps every registered dataset's latest version; `--strict` exits non-zero on any finding. |
| `distill --session <id> -o <ds.jsonl>` | Turn ratings on a session's turns into an eval dataset + one grader. Flags: `--all-sessions`, `--graders-out <g.yaml>`, `--min-score`, `--judge [--judge-model <m>]` (emit an `llm_judge` grader instead), `--register <name>` (also promote a new dataset version into the registry), `--no-redact`. Distilled samples are stamped `metadata.source: "production_log"` with the rating channel preserved as `metadata.feedback_source`. |

**Provenance taxonomy.** `metadata.source` is canonical and enforced, with
exactly five members: `human_authored`, `production_log`, `synthetic`,
`synthetic_human_verified`, `canary`. (`dataset mine` stamps
`production_log` with `mined: true` alongside — there is no `mine` source.)
Registering a dataset whose declared provenance falls outside the taxonomy
**warns** on stderr and lists the offenders; it never fails. The registry
does enforce one hard invariant, though: a `synthetic` sample carrying an
`expected_output` is **refused**, with a pointer at
`synthetic_human_verified`.

### Graders and judges

| Command | Purpose |
| --- | --- |
| `graders suggest [-o <file>] [--runs <dir>\|last:N] [--model <m>] [--spec <n>] [--min-score F] [--force]` | Draft grader suites from failure rationale: cluster `grades.json` rationale (via the run-history index), judge criterion scores, and rating comments into themes; draft deterministic graders per theme (+ an `llm_judge` rubric with `--model`) into a **review file** — never auto-applied. |
| `graders test --graders <g.yaml> --golden <verdicts.jsonl> [--judge-model <m>] [--min-agreement F]` | Meta-eval the graders themselves — *are the instruments any good?* The golden file is strict JSONL, one `{id, input, agent_output, expected_passed, expected_score?}` per line (stray keys and duplicate ids are loud, line-numbered errors; `expected_score` normalizes to 0..1). Deterministic and `type: registry` graders (pack `opts` included) replay **credential-free**; `llm_judge` graders need visible judge credentials and are **skipped with a clear notice** without them, while the rest still test. `target: transcript` judges always skip — a golden verdict carries only the final output. Per tested grader it reports the agreement rate, **Cohen's kappa** against `expected_passed`, FP/FN counts with up to 5 exemplar ids each, abstained/error counts (excluded from the agreement denominator), and mean absolute score error when `expected_score` is present. `--min-agreement F` exits non-zero when any **tested** grader falls below the floor. Judge rubrics test at their declared `passing_score` (default 3/5); the `judge calibrate --apply` overlay is deliberately **not** applied, because the meta-eval measures the graders file as written. `--judge-model` defaults to `claude-sonnet-4-5`. |
| `graders card (--graders <g.yaml> \| --template <family>) [-o <file>]` | Render a graders file as its markdown **rubric card**: per-grader type, `opts` and thresholds; per-`llm_judge` rubric criteria + anchors (or labels), the passing cut, and panel/repeats/temperature/target; plus the config's `gradersHash` — the same identity eval-run history records. Deterministic (content-derived, no timestamps), so carding a template family and then carding the scaffolded copy proves the copy is unedited. `--template` cards an eval-template family without scaffolding it first. Stdout by default. |
| `judge calibrate [--graders <g.yaml>] [--dataset <d>] [--model <m>] [--sessions N\|all] [--apply]` | Calibrate an `llm_judge` against human ratings: correlation / bias / ROC-optimal cut point over paired `(human rating, judge score)` samples. `--apply` writes the calibrated `--min-score` default to `.crewhaus/judge-calibration.json`, now **atomically** (temp file + rename — a torn file had been silently mis-gating whole runs). `--dataset` is real as of v0.4.x (it was accepted, documented in `--help`, and never read): it **adds** calibration pairs from the golden verdicts a distilled dataset carries, on top of the session-ratings pairs. A sample pairs when `metadata.user_rating` is a number in [0,1] **and** `expected_output` is the non-empty answer that rating was placed on; `metadata.correction` and `metadata.gold_refreshed` samples are skipped as mis-paired, and samples already paired from scanned sessions are dropped as duplicates. A `--dataset` yielding zero usable pairs dies loudly with the contract spelled out. Bare registry refs resolve train+dev; the locked test split stays locked. With a `--graders` file whose only `llm_judge` entries are **categorical**, calibrate explains that a label-gated rubric has no scalar cut to calibrate. |

**Judge temperature is now pinned to 0 by default** (previously the
provider default, ~1.0). `llm_judge` verdicts — and therefore
`(spec, dataset)`-keyed baselines — may shift on the first run after
upgrading. Override per grader with a rubric-level `temperature:`. The
adapters that have a native control all map it, and each omits the pin
where its API rejects an explicit temperature: Anthropic and
Anthropic-on-Bedrock (dropped when extended thinking is enabled, and for
Claude Opus 4.7+ and all Claude 5-family models — Sonnet 5, Opus 5,
Fable 5 — which reject the parameter outright), OpenAI (dropped for
o-series/gpt-5 reasoning models, which reject a non-default temperature),
Gemini, and Bedrock Converse. If a provider the adapters don't know
rejects the pin anyway, the judge retries that call once without it. Net:
the pin holds wherever the model accepts it, and judging works everywhere
else instead of failing the run.

The `graders.yaml` grammar itself grew — a top-level
`combine: all|any|weighted` with `passing_threshold`, per-grader `weight`, a
new `expected_contains` grader, categorical `llm_judge` rubrics, judge
`judges:` panels / `repeats:` / `target: transcript`, and validated `opts:`
on `type: registry` entries. Both the top level and the `llm_judge` and
`registry` entries are now **strict**, so a key that used to be silently
stripped is a loud parse error. The grammar reference lives in
[Recipe 12 — Eval harness](https://github.com/crewhaus/demos/blob/main/walkthroughs/12-eval-harness.md)
and
[Recipe 34 — Building custom graders](https://github.com/crewhaus/demos/blob/main/walkthroughs/34-building-custom-graders.md);
`crewhaus graders card` prints what a given file actually declares.

---

## The observer/advisor

Suggestions that reach beyond the prompt — mined from durable session
telemetry, validated against the eval gate before anything is applied. The
advice feed also has a console surface: the Hangar's
[Advisor](#the-hangar) folds it — alongside preflight, spec-lint,
eval-health, cost, incident and parked-approval signals — into one
severity-ranked feed whose quick actions queue CLI verbs from a closed
vocabulary, each shown with its CLI twin.

| Command | Purpose |
| --- | --- |
| `advise [--session <id> \| --all] [--json] [-o <dir>]` | Mine session logs + audit records for typed `SpecPatch` suggestions; writes `suggestions.json` + `report.html` (default `.crewhaus/advice`). Feed the result to `optimize --from-advice`. |
| `tools list` | List every builtin tool + its metadata. |
| `tools suggest [spec]` | Rank builtins against `agent.instructions` (keyword match). |
| `tools audit [--sessions N\|all] [--json]` | Mine `tool_stats` vs. grants — unused / chronically-failing tools, learned read-only flags. |
| `permissions suggest [--apply] [--sessions N\|all] [--json]` | Mine ask/deny history into `settings.json` rules (the `fewer-permission-prompts` analogue). `--apply` is interactive-confirm only — permissions are never eval-gated and stay outside the optimizer by design. |

---

## Model and cost automation

Provider failover, market scanning, right-sizing, and budget caps. Model
fields sit outside `OPTIMIZABLE_PATHS`, so the write paths here are direct
comment-preserving CST edits, always human-initiated.

| Command | Purpose |
| --- | --- |
| `model-scan [--dataset <d>] [--graders <g>] [--limit N] [--same-provider] [--write]` | Scheduled market scan: enumerate capability-compatible replacements for `agent.model`, eval each on the spec's dataset, and emit a proposal (+ `patch.json`) when a candidate beats current on score at lower cost. Proposal-only unless `--write`. Also: `--concurrency`, `--seed`, `--judge-model`, `-o`. |
| `model right-size <spec> [--min-cost-drop F] [--pass-rate-tolerance F] [--per-slot-limit N] [--write]` | Downshift search: swap one model slot at a time to cheaper pricing-table siblings, score with the eval fitness, recommend only when the pass rate holds. Proposal-only unless `--write`. |
| `pricing sync [--file <feed>]` | Load a versioned pricing feed into `~/.crewhaus/pricing/` so cost tracking overrides the hand-snapshotted table without a code release. |
| `pricing show` | Print the active pricing table. |
| `cost-summary --session <id> [--tenant <t>] [--format <f>]` | Aggregate `cost_accrual` events into total USD spend, including per-session cache-hit ratio and realized cache savings. |
| `route status [--dir <root>]` | Show the adaptive `agent.model_pool` reward scoreboard: per-difficulty-band arms with sample count, mean reward, mean latency/cost, best-per-band starred (what a `learned` policy exploits). |
| `route explain <session> [--dir <root>]` | Replay one run's per-turn `model_pool` routing decisions from its persisted `model_route` events — turn, difficulty band, model, policy, explore/exploit, and reason. (v0.2.2) |
| `route reset [--dir <root>]` | Wipe the `model_pool` reward scoreboard (kill switch). `--dir` points at the `.crewhaus` root (default `.crewhaus`). |

Model automation is also declarative in the spec:
`agent.model_fallbacks` + `agent.circuit_breaker` (failover chain),
`agent.model_tiers` (two-tier turn router), `agent.model_pool` (N-candidate
adaptive routing whose `learned` policy improves with usage — since v0.2.2 it
also **explores online** [ε-greedy or Thompson sampling] and works on the
pipeline/research/batch/browser shapes as well as cli/channel/managed; mutually
exclusive with the previous two), and `budget:` (run-level spend cap). Adaptive
routing also applies to the interpreted `crewhaus run` path, not just compiled
bundles. See the [spec reference](GETTING-STARTED.md#model-cost-and-budget-blocks).

---

## Feedback, memory, and knowledge

Capture ratings, harvest golden examples, and let a harness get smarter from
its own history.

| Command | Purpose |
| --- | --- |
| `rate --session <id> [--turn N] (--thumbs up\|down \| --stars 1-5 \| --score 0-1) [--comment <t>] [--rater <id>] [--adjudicate]` | Rate an assistant turn. `--adjudicate` marks the record as an **adjudication**: at distill time it always wins a multi-rater disagreement on that turn and closes it. |
| `feedback --session <id> --text <msg> [--turn N] [--correction <better answer>] [--rater <id>] [--adjudicate]` | Attach a comment or a correction (a better answer) to a turn. On this surface `--adjudicate` **requires** `--correction` — a comment alone carries no verdict. |
| `review list [--kind <k>] [--all]` | The persistent human-review queue at `.crewhaus/review/queue.jsonl` (append-only). Kinds are exactly `abstained \| needs_review \| rater_disagreement \| quarantine`. Open items only by default; `--all` includes resolved ones. Three feeders write it, all idempotent (entry ids are deterministic from the source key): `crewhaus eval` enqueues judge-abstained and panel-entropy samples at run end (best-effort — a queue write can never fail the run), `distill` enqueues unresolved rater disagreements, and `dataset mine` enqueues pointers to quarantined candidates. |
| `review next [--kind <k>]` | Take the oldest open item with its context. **In a TTY** it records your verdict, routing a session-turn item through the same capture machinery as `crewhaus rate` and recording it as an adjudication, so the disagreement closes at the feedback source too. **In a non-TTY it prints the item and exits** — it never hangs a script or a CI pipe. |
| `review resolve <id> [--note <t>]` | Close an item non-interactively, with an optional recorded note. |
| `fewshot harvest [--all-sessions] [--min-score F] [-o <pool>]` | Harvest up-rated turns into a golden few-shot pool (PII/secret-redacted); consumed by `optimize --few-shot`. |
| `fewshot show [--k N]` | Print the pool as the injectable prompt block. |
| `faq distill [--sessions N\|all] [--min-score F] [--min-occurrences N] [-o <skill-dir>]` | Cluster recurring user questions into an auto-discovered FAQ skill under `.crewhaus/skills/faq/`. |
| `lessons update [--sessions N\|all] [--low-score F] [-o <LESSONS.md>]` | Mine corrections + failure→fix patterns into a deduped, auto-loaded `LESSONS.md`, plus per-user preference files under `.crewhaus/preferences/`. |
| `sessions summarize [--before <date>] [--evicted [--ttl-days N]]` | Fold sessions into a durable index; `--evicted` indexes each session just before TTL eviction deletes it (the summarize-before-evict hook). |
| `sessions tail [<session>] [--dir <root>] [--no-follow] [--interval <ms>]` | Follow a session's transcript live — a `tail -f` for a running agent (the per-turn view `crewhaus dev` points at). With no `<session>`, tails the most-recently-updated one under `.crewhaus/sessions`; each user/assistant turn, tool call + result, and failure prints one line as it lands. `--no-follow` dumps the current transcript and exits (scriptable/CI); `--interval` sets the poll ms (default 500). |
| `sessions export --format trajectories [--out <file.jsonl>]` | Emit one JSONL line per agent step — a `(state, action, observation, reward)` tuple — from every session event log under `.crewhaus/sessions`. `reward` is terminal-sparse (null except the last step, which carries the last `eval_graded` score, else the latest user rating normalized to `[0,1]`, else null; `rewardSource` says which rung fired). `--out` writes to a file; omitted, it streams to stdout. **Experimental (G53):** inference-time scaffolding (eval → optimize → flywheel) stays the mature improvement lane — this export exists so an external trainer can consume real sessions, and is deliberately **not** wired into `crewhaus optimize`. |
| `knowledge sync [--pull \| --push] [--root <dir>] [--shared <dir>] [--dry-run] [--no-redact]` | Cross-harness knowledge sync: publish/import shared memories, distilled grader suites, and optimizer-winning instruction fragments to a versioned shared store. Redacted by default. |

**Multi-rater agreement.** Feedback stays append-only — every rater's
record is kept — and `crewhaus distill` now resolves a multi-rater turn
explicitly instead of letting the later timestamp win: all-thumbs turns
resolve by **majority**, stars/scale (or a mixture) by the **mean
normalized score**, and an `--adjudicate` record always wins and closes the
disagreement. A true split verdict with no adjudication is **not** silently
labeled — the turn is withheld from the dataset and enqueued for human
review. Multi-rater samples record every rater's normalized verdict in
`metadata.ratings` (plus `metadata.adjudicated` when an adjudication settled
it), and `distill` prints per-turn agreement and an overall **Cohen's
kappa** whenever any turn has ≥ 2 raters. Single-rater corpora — including
everything recorded before this release — distill byte-identically.

**PII redaction at ingestion (behaviour change).** `crewhaus distill`, the
unattended `feedback.autoDistill` teardown, and `crewhaus dataset mine` now
run the same PII/secret detector set over every free-text field at sample
construction, deterministically replacing hits with `[REDACTED:<kind>]` and
leaving non-PII text byte-identical. `--no-redact` opts out on `distill`,
`dataset mine`, and `optimize --ratings`; the `autoDistill` teardown is
unattended and **always** redacts.

Auto-capture/recall is also a declarative spec block — `memory:` wires the
Remember/Recall tools and the capture/recall passes; `feedback:` configures
the capture surfaces and `autoDistill`/`exitPrompt`. `feedback:` now also
parses on `target: managed`, where the daemon serves a `feedback.submit`
JSON-RPC method appending to `.crewhaus/feedback/<tenant>.jsonl` — the exact
sink `distill` / `optimize --ratings` / `judge calibrate` already read. On
both the **channel** and **managed** daemons, `autoDistill: true` now
registers a `feedback_distill` janitor step that runs on the daemon's own
clock — a daemon never reaches a `crewhaus run` teardown — with the same
"≥ 5 unprocessed ratings" trigger and watermark as the CLI path
(`CREWHAUS_AUTODISTILL_THRESHOLD` overrides the trigger;
`CREWHAUS_AUTODISTILL=0` disables the tick entirely). `exitPrompt`
(no REPL to exit) and `channelReactions` (the channel shape's own inbound
gate) parse there for schema uniformity and emit a compile warning rather
than being silently honoured. See the
[spec reference](GETTING-STARTED.md#memory-and-feedback-blocks).

---

## Self-healing operations

Canary deploys, drift sentinels, MCP health, incident bundles, and load
testing — the operational safety net for a running harness.

| Command | Purpose |
| --- | --- |
| `deploy canary <spec> <version> --traffic 5,25,50,100 --dataset <d> --graders <g>` | Eval-gated ramp with auto-rollback: register a candidate, step traffic, eval both versions per step, gate on the regression runner, auto-promote or auto-rollback — all audit-logged. Also: `--env`, `--name`, `--from`, `--concurrency`, `--seed`, `--judge-model`, `--max-pass-rate-drop`, `--max-p95-latency-ms`. (Before v0.4.x this whole flag block was misfiled on `propose`'s arg schema, so every documented canary invocation died at parse with `unknown flag: --dataset`. It parses now.) |
| `deploy canary … --allow-test-split` | Canary is a release gate and therefore a *sanctioned* spender of the held-out split: an explicit `#test` dataset ref is consumable here, behind the same opt-in `crewhaus eval` uses. |
| `deploy canary … --traffic-split [--experiment <name>] [--experiment-dir <d>]` | Also write a deterministic variant assignment plus a per-version eval ledger into the experiment store. **Boundary: this does not split live traffic.** Nothing routes requests; it records which version a given request key *would* map to and accumulates per-version outcomes, and the assignment is removed when the ramp concludes. |
| `mcp doctor [--probe] [--sessions N] [--format json\|table]` | Per-server MCP health scoring from `mcp_call` events, tool-schema drift watch (`--probe` does a live `listTools`), and the runtime auto-quarantine decision. |
| `incident collect --session <id> [--kind <k>] [--reason <r>] [-o <dir>]` | Assemble an incident bundle from a session's traces + audit + cost + `doctor` output. |
| `failures report [--sessions N\|all] [--propose-taxonomy] [--dir <root>] [-o <file>] [--json]` | Aggregate and cluster `run_failed` events + incident records by failure class and message similarity across recent sessions. `--propose-taxonomy` drafts `failure_taxonomy` spec entries from the clusters (reusing the advise drafting machinery + specificity floor) to stdout or `-o`. |
| `loadtest <spec> [-c N] [-n N] [--duration <d>] [--rps N] [--gate] [--max-p95-ms N] [--max-error-rate F] [--stub-latency-ms N] [--format <f>] [-o <dir>]` | Replay a dataset against a locally-booted daemon at N concurrent sessions: p50/p95, TTFT, breaker/rate-limit trips, cost per request. `--gate` exits 1 when p95 latency / error rate exceed the thresholds. |
| `intents [--sessions N\|all] [--top N] [--format <f>] [-o <dir>]` | End-user intent analytics digest: clustered top intents, low-rated / no-answer clusters, tool-failure-correlated requests, week-over-week trends. |

The `observability.slo:` spec block declares production SLOs and the
mitigation ladder the runtime walks on a sustained breach; see the
[spec reference](GETTING-STARTED.md#observability-and-slo-block).

---

## Deploy, promote, and govern

Re-pin specs across environments, and route prompt changes through review
like code.

| Command | Purpose |
| --- | --- |
| `spec put\|list\|get\|pin\|alias\|log <name> …` | Versioned spec storage. `spec log <name>` renders the per-spec changelog (field-level diffs + optimizer provenance). Flags: `--root-dir`, `--tenant`. |
| `deploy promote <name> <fromEnv> <toEnv> [--require-approval] [--check-pr]` | Copy an environment's spec pin, with an audit-log entry. `--require-approval` refuses to flip a protected env's pin (per `.crewhaus/environments.json`) until an approval quorum is met; `--check-pr` consults a green proposal PR as an approval witness. Flags: `--root-dir`, `--tenant`, `--actor`. |
| `deploy rollback <name> <env> <version> [--require-approval] [--check-pr]` | Re-pin an environment to a specific version. |
| `deploy fly\|render\|railway\|heroku <spec> [-o <dir>] [--app <n>] [--image <ref>] [--region <r>] [--live]` | Scaffold PaaS deploy manifests for a daemon (channel/server) shape. Scaffold-only by default; `--live` (gated on the provider token) drives the `cloud-adapter-*` engine's API deploy. Provider-specific live inputs: `--org` (fly org slug), `--project` (railway projectId), `--owner` (render ownerId). Shares the `deploy` verb with the registry-pin `promote`/`rollback`/`canary` actions but nothing else. |
| `deploy canary …` | See [Self-healing operations](#self-healing-operations). |
| `experiment status [--name <n>] [--control <version>] [--min-n N] [--json] [--dir <root>]` | Per-version outcome and rating deltas with **Wilson 95%** intervals, folded from the append-only ledger under `.crewhaus/experiments/<name>.jsonl`. Refuses to name a winner while any version sits below `--min-n` (default **30**) and says which. Repeat **EVAL** measurements of the same `(version, dataset sample)` are collapsed — re-running a ramp re-measures fixed samples, and counting those as independent would narrow the intervals on evidence that did not grow; serving/CLI records are never collapsed. `--json` carries the per-version `sources` breakdown and the `boundary` note alongside the numbers. |
| `experiment record --name <n> --version <v> --outcome pass\|fail [--score F] [--rating F] [--key K] [--source <s>] [--dir <root>]` | Append **one** outcome for a version — how a serving integration reports back. |
| `experiment assign --name <n> --key <request-key> [--dir <root>]` | Print the version a request key deterministically maps to (sha256 bucket — the same hash the canary controller's `route()` uses), stable across processes with no shared state. |
| `propose <proposed.yaml> [--current <spec>] [--source optimize\|advise\|model-scan\|manual] [--as-version <v>] [--dry-run]` | Package a spec change into a review artifact and open a PR (the governance wrapper around optimize/advise write-back). Never auto-merges. Also folds in optimize provenance via `--optimize-dir`, `--run-id`, `--score-before`/`--score-after`. |

**What `experiment` is, and is not.** It is the honest subset of an online
A/B test CrewHaus can actually reach: **selection** (a stable request key
always maps to the same variant version, in every process, with no shared
state) and **accounting** (an append-only per-version outcome ledger folded
into success rates, mean scores/ratings and their deltas). It does **not**
intercept live requests. The numbers are outcomes *reported* against each
version — `deploy canary --traffic-split` eval samples and/or
`crewhaus experiment record` — so this is not a live-traffic experiment
unless your own serving boundary made it one. `status` prints that boundary
note with every report, and `--json` carries it in the payload, precisely so
a verdict cannot be quoted onward without it.

`compile` and `optimize --write-back` auto-register the changed spec into
the local registry with a distilled changelog; `compile --no-register` /
`optimize --no-register` opt out.

---

## Fleet, lifecycle, and marketplace

Operate more than one harness, share what they learn, and decommission
cleanly.

| Command | Purpose |
| --- | --- |
| `fleet list [--root <dir>]` | Cross-harness inventory (any directory carrying a `crewhaus.yaml`; discovery skips `.crewhaus/`, `node_modules/`, `.git/`, `dist/`, `.worktrees/`). |
| `fleet status [--root <dir>]` | Per-harness health rollup. |
| `fleet run <sub> [--filter <glob>] [--root <dir>] [--allow-mutating] [--yes]` | Bulk-run a subcommand across the filtered fleet. Read-only bulk subcommands (`eval`, `doctor`, `security digest`, `audit verify`) run freely; a mutating subcommand requires `--allow-mutating` and per-harness confirmation. |
| `retire <spec> [--archive <dir>] [--dry-run] [--force] [--push-knowledge [--shared <dir>]]` | Audited harness decommissioning: archive sessions/feedback/memories, retention-purge, rotate-then-revoke secrets, tombstone registry entries, emit a final compliance bundle. `--force` retires despite an active pin; `--dry-run` prints the plan. |
| `plugins list\|search\|install\|uninstall\|outdated\|publish …` | Marketplace plugins CLI. `search -q <text>`, `install <name> [--version <v>]`, `publish --manifest <plugin.json>` (opens a publish PR). Flags: `--registry <dir\|url>`, `--plugins-dir`, `--allow-unsigned` (dev only), `--dry-run`. |
| `templates list\|search\|use …` | Marketplace templates CLI. `search -q <text> [--target <t>]`, `use <name> [--into <dir>] [--subdir <s>]` scaffolds a template into a workspace. A manifest may now declare `kind: grader-template` plus an `evalAssets` block (a `graders.yaml`, reviewer notes, and an optional seed dataset); `templates list` marks such a manifest **`[eval-template]`**. Both fields were appended to the canonical signing payload **append-only**, so a manifest declaring neither serializes byte-identically and existing signatures keep verifying. **Not yet wired for consumption:** `templates use` refuses an eval-asset template, and `scaffold-evals --template` resolves only the CLI's embedded first-party families — it performs no registry fetch and never reads `CREWHAUS_TEMPLATE_REGISTRY`. |

---

## The hangar

The harness manager: a local web console over every harness registered on
this machine (`hangar`), the registry that backs it (`harness`), and the
terminal twin that supervises one harness with no console running
(`daemon`). Full reference — the security model, the on-disk layout,
`crewhaus.control.v1`, and troubleshooting — in [HANGAR.md](HANGAR.md).

The console's screens include the **Advisor**: a per-harness tab and a
fleet board (`#/advisor`) folding every alert, suggestion and optimization
signal the manager can derive into one severity-ranked feed
(`critical | warn | suggestion`), plus a trend view, generated reports
(model-usage / costs / usefulness / optimization), and an issue inbox that
turns a described problem into queued CLI work (`optimize` by default) or a
recorded note. Nothing in the feed is invented, and a suggestion is never
an application — a quick action queues a CLI verb from a closed vocabulary,
its CLI twin shown beside the button, or deep-links the screen that owns
the fix. A decision is a record, not a deletion: acting takes an optional
comment, dismissing requires a reason, both append to
`<harness>/.crewhaus/advisor/decisions.jsonl`, and every dismissal can be
reopened; zero open items renders as "running optimally", not an empty
screen. The verbs the quick actions queue are the CLI's own — see
[the observer/advisor](#the-observeradvisor).

One property explains the shape of all three: **supervision state is
harness-local**, under `<harness>/.crewhaus/run/`. Nothing about a running
daemon lives in a central manager directory, so `crewhaus daemon` needs no
server, and a daemon either head starts is adopted by the other.

`--help` / `-h` works at the **verb** position only in all three families
(`crewhaus daemon start --help` is an unknown-flag error). Bare
`crewhaus harness` and `crewhaus daemon` print usage; **bare
`crewhaus hangar` boots a server.**

### `crewhaus hangar`

| Command | Purpose |
| --- | --- |
| `hangar [serve]` | Boot the manager console over the machine-wide registry and open it in the browser. Bare `crewhaus hangar` — and any invocation whose first argument starts with `--` — is `hangar serve`. |
| `hangar serve --port <n>` | TCP port; default `4200`, `0` for an OS-assigned one. Must be an integer 0..65535 in canonical spelling (`007` and `80.0` are refused). |
| `hangar serve --host <h>` | Bind interface; default `127.0.0.1`. Implies auth **and** requires `CREWHAUS_HANGAR_ALLOW_REMOTE=1` for any non-loopback value. Loopback is `localhost`, all of `127.0.0.0/8`, and `::1` in any spelling; `0.0.0.0` and `::` are wildcards, **not** loopback. Hostnames are never resolved — an unrecognised host fails closed. |
| `hangar serve --no-auth` | Disable the bearer token. Loopback-dev only; refused together with `--host` or `--smoke`. |
| `hangar serve --no-open` | Do not spawn the browser. |
| `hangar serve --read-only` | Boot with every mutating route refused (403). The screen-share posture; it can still be lifted from the UI. Not persisted — a normal restart gives a writable console back. |
| `hangar serve --read-only-locked` | As above, and the mode cannot be lifted over the wire — only by restarting without the flag. **Implies `--read-only`.** |
| `hangar serve --smoke` | Boot on an ephemeral port, run four self-checks (healthz, the embedded UI shell, `/api/harnesses` with a token → 200, without → 401), then exit. The release workflow's compiled-binary smoke entry. Refused with `--port` or `--no-auth`. |
| `hangar status [--json]` | Lock / port / registry / token report. Reads the lock file, not the socket, so it works with no server running. **Always exits 0.** |
| `hangar open` | Trade the token file for a fresh single-use boot ticket and open the running console. Takes no flags. **Exits 1** when nothing is running. |

The console binds `127.0.0.1:4200` by default and hands its bearer token to
the browser as a URL `#fragment` via a single-use `/boot/<nonce>` path — the
token is never a command-line argument. One instance per hangar root, held
by `<hangarRoot>/hangar.lock`; a stale lock from a dead pid is replaced
automatically with a note. Ctrl-C stops attached runs, **leaves daemons up**
(they are adopted by the next boot), and releases the lock.

### `crewhaus harness`

The registry verb family — machine-wide, wherever the directories live.
`run`, `compile`, `eval` and `dev` self-register the harness they touch
(origin `run-hook`), so the list fills itself from normal use.

| Command | Purpose |
| --- | --- |
| `harness list [--group <name>] [--json]` | Every registered harness, joined with the on-disk inventory. Missing directories are flagged, never auto-pruned. |
| `harness show <dir\|hrn_id> [--json]` | One harness: registry entry + inventory row + health rollup. |
| `harness add <dir>` | Register a directory (origin `manual`). Warns rather than fails when there is no readable `crewhaus.yaml`. |
| `harness remove <dir\|hrn_id>` | Drop the registry row only — the directory itself is untouched. |
| `harness relocate <hrn_id> <newDir>` | Point an entry at a moved directory, keeping its id. Throws when the new directory already belongs to another entry. |
| `harness group <name> [--add <dir\|id>] [--remove <dir\|id>] [--color <c>] [--order <n>] [--member-order <n>] [--list]` | Bare form creates the group; `--add`/`--remove` manage membership and are mutually exclusive. `--order` orders the **group** among groups; `--member-order` (which requires `--add`) orders **one member inside** it — the boot order `daemon start --group` walks. Both must be positive integers. `--list` prints the resulting walk. |
| `harness tag <dir\|id> (--add <tag> \| --remove <tag>)` | Exactly one of the two is required. |
| `harness pin <dir\|id> [--off]` | Pinned entries survive prune prompts. |
| `harness scan [--root <dir>]` | Discover harnesses (directories carrying a `crewhaus.yaml`) under the given root — or all configured scan roots — and upsert each (origin `scan`). An explicit `--root` is **remembered** as a scan root. Scan never prunes vanished rows; it counts them. |
| `harness preflight <dir\|id> [--json]` | Typed will-it-boot checks across nine areas (spec, env, credentials, channels, mcp, ports, bundle, durability, hooks). Runs against the **merged spawn env**, so a credential the daemon reads from `.env` is not reported missing. **Exits 1 on any blocking finding.** Works against an unregistered directory. |

The registry is one JSON file, `<registryRoot>/harnesses.json` (format v2),
under `CREWHAUS_REGISTRY_ROOT` (default `~/.crewhaus`). Ids are `hrn_` +
16 hex and stay stable across renames; the absolute directory is the upsert
identity key. `CREWHAUS_NO_REGISTRY=1` turns every write into a no-op —
each mutating verb then says so on its last line.

`harness` is **registry-centric**; [`fleet`](#fleet-lifecycle-and-marketplace)
is the **filesystem-centric** twin that walks a `--root` and needs no
registration. Both are supported indefinitely; neither replaces the other.
The one place they meet is `fleet --group <name>`, which reads membership
from the machine registry.

### `crewhaus daemon`

Supervise one harness from the terminal, driving `@crewhaus/harness-supervisor`
directly — no console required. Every verb takes an optional first
positional `[<dir|hrn_id>]`; with none, the harness is the current directory.

| Command | Purpose |
| --- | --- |
| `daemon start [<dir\|hrn_id>] [--force] [--ack <id,id>] [--no-preflight] [--compile\|--no-compile] [--group <name>] [--parallel]` | Prep, preflight, then spawn. `--force` waves through every **forceable** blocking item; `--ack` waves specific ones through by stable id. Missing channel secrets can **never** be forced — the compiled daemon exits 2 on exactly that set. Preflight runs against the merged spawn env (shared env files, then the harness `.env` chain, under `process.env`), not `process.env` alone. `--compile` recompiles **only when the spec is newer than the bundle**, then runs `bun install --cwd <bundle>`; with neither flag the harness's `manager.autoCompile` decides. Exits 1 when already running, when preflight or an operator hook refuses, or when the spawn plan cannot be built. |
| `daemon submit [<dir\|hrn_id>] --brief-file <path> [--force] [--ack <ids>] [--no-preflight] [--compile]` | Run ONE pipeline with a brief on stdin — the supervised path for `crew`, whose input is a document. Tracked in the run ledger as a job and **never restarted**. The brief travels as a path (kernel-fed stdin), never in argv. A missing or empty brief, or a shape whose input is not a brief, is refused up front. |
| `daemon restart [<dir\|hrn_id>] [--force] [--ack <ids>] [--no-preflight] [--compile\|--no-compile] [--group <name>] [--parallel]` | Stop, then start. The spawn plan is rebuilt, so a recompile between the two is picked up. |
| `daemon stop [<dir\|hrn_id>] [--grace <ms>] [--group <name>] [--parallel]` | SIGTERM, then SIGKILL after the grace (default `15000`). Exits 1 only when a live daemon holds the runfile but was not adopted, so nothing was signalled. |
| `daemon status [<dir\|hrn_id>] [--json]` | Runfile / liveness / control port / last 5 runs, plus `would run: <plan>` — the paste-able shell equivalent (the env is omitted, because it carries the control token). Always exits 0. |
| `daemon logs [<dir\|hrn_id>] [--tail <n>] [--follow] [--run <run_id>]` | The **scrubbed** capture of a run, never the raw `logs/<runId>.log`. `--tail` defaults to 40 and may reach back at most 512 KiB; `--follow` polls every 500 ms and exits when the pid dies, reading once more afterwards so a crash's last lines are not dropped. An unknown `--run` exits 1. |
| `daemon wake [<dir\|hrn_id>] --lane heartbeat\|schedule [--reason <r>]` | One synthetic tick down the daemon's OWN timer path, via `crewhaus.control.v1` — the same code the schedule fires. `--lane` is required. |
| `daemon drain [<dir\|hrn_id>]` | Stop intake, finish in-flight work, exit 0. Falls back to SIGTERM when the control plane is unreachable. Takes no flags. |

`wake` and `drain` refusals that are **facts about the bundle** —
`no_control_port`, `lane_not_armed`, `draining` — exit **0**, not 1: a spec
that declares no schedule lane is an answer, not an error. `tick_in_flight`
is the only retryable code.

`--group <name>` on `start`, `restart` and `stop` walks every member of a
registry group in its declared boot order (**reversed** for `stop`), keeps
going past a member that refuses, skips shapes with no daemon *with a note*,
prints a per-member summary, and exits non-zero if any member failed.
`--parallel` opts out of the ordering where it does not matter.

Per-harness prep lives in the harness's own `.crewhaus/settings.json` under
`manager`: `envFiles` (shared fleet env files, merged **under** the local
`.env` chain), `autoCompile`, and `hooks.postCompile` / `hooks.preSpawn` —
operator commands run between compile and spawn, whose non-zero exit refuses
the start exactly like a blocking preflight finding. A string declaration is
one command and is not word-split; an array is an argv vector. Nothing goes
through a shell. See [HANGAR.md](HANGAR.md#per-harness-prep-compile-and-hooks).

Exit codes 20 (spec), 21 (config), 30 (auth), 31 (billing) and 33
(crewhaus_budget) are **terminal** and never auto-restarted; 36 is *parked*,
waiting on `approvals grant`. Everything else backs off 500 ms → 30 s with a
cap of 5 restarts per rolling 10 minutes, then `crash-looping`.

---

## Safety that learns

Turn accumulated security signal — blocked attacks, egress verdicts,
redaction history — into tuned defenses.

| Command | Purpose |
| --- | --- |
| `security digest [--since 7d\|30d\|ISO] [--format text\|json\|html] [-o <dir>] [--notify <url>] [--dir <root>]` | Triage rollup of denials / egress verdicts / injection hits / breaker flips over a window. `--notify` POSTs the JSON digest to a webhook. |
| `security corpus [--since <w>] [--dir <root>] [--min-support N] [--json]` | Build a replay corpus from quarantined blocked payloads. |
| `security corpus check [--dir <root>]` | Run the security regression corpus (for CI on every detector change). |
| `egress review [--dir <root>] [--propose [-o <file>]] [--json]` | Cluster egress/boundary history by `(origin, sink, lineage kind)` into learned security suggestions. `--propose` writes a `suggestions.json` that `optimize --from-advice` can consume. |
| `pii tune [--sessions N\|all] [--dir <root>] [--secret <hmac-key>] [--write] [--json]` | Propose PII allowlist entries from hashed redaction history (a value seen identically N+ times is likely a fixed identifier, not PII). `--write` persists `<dir>/.crewhaus/pii-policy.json`. The `--secret` must match the redactor's HMAC key. |
| `justification calibrate [--sessions N\|all] [--dir <root>] [--json]` | Flag 100%-allow tools (drop `requireJustification`) and chronic-deny tools (`alwaysDeny` candidates) from the outcome proxy. |
| `justification preflight <spec> [--goal <g>] [--json]` | Replay historical `(tool, justification)` triples through a candidate judge and report verdict flips **before** switching judge config. |
| `onchain tune <spec> [--history <f>] [--cap-margin N] [-o <file>]` | Mine wallet receipts into a proposed `transaction_policy` SpecPatch (per-contract allowlists, value caps). |
| `onchain sentinel <spec> [--history <f>] [--baseline <f>] [--max-multiple N] [--json]` | Flag anomalous spend: transactions exceeding N× the per-contract max. |

---

## Compliance, audit, and retention

Scheduled evidence collection, tamper verification, and GDPR/TTL enforcement.

| Command | Purpose |
| --- | --- |
| `compliance evidence (--framework <id> \| --all-frameworks) [--control <id>] --period <p>\|current [--audit-dir <d>] [--out-dir <d>] [--signing-key-env <ENV>] [--fail-on-empty]` | Collect SOC 2 / ISO 27001 / HIPAA evidence bundles from the audit log. `--period current` resolves the current UTC quarter (a cron never hardcodes a stale label); `--fail-on-empty` exits 1 on a zero-evidence control (for scheduled runs). |
| `audit verify [--dir <auditDir>] [--anchor file:<path>]` | Verify the hash-chained audit log (tamper check) with exit codes; `--anchor` cross-checks an append-only file anchor store. |
| `retention sweep [--dry-run] [--dir <root>]` | Scheduled GDPR/TTL enforcement over `.crewhaus` stores (sessions expire per `.crewhaus/retention.json`; audit is export-only). |
| `retention export <outDir> [--since <date>] [--dry-run] [--dir <root>]` | Right-to-export: copy session/audit records out. |
| `retention purge [--before <date>] [--dry-run] [--dir <root>]` | Right-to-delete: purge expired records now. |

---

## Registry, secrets, and state

| Command | Purpose |
| --- | --- |
| `secrets doctor` | List known secrets via the configured backend (`--backend env-var\|file`). |
| `secrets rotate <name> [--value V]` | Rotate a named secret (file backend). |
| `state backup [-o <file.tar.gz>] [--exclude <glob,glob>]` | Snapshot the cwd `.crewhaus` state dir (sessions, feedback, memories, datasets, durable-state) to a tarball. |
| `state restore <file.tar.gz> [--into <dir>] [--force] [--merge feedback\|all]` | Restore a snapshot (refuses a non-empty `.crewhaus` without `--force`). `--merge feedback` folds deployed-bot feedback back into the dev loop (deduped). |

---

## Distribution and infrastructure

| Command | Purpose |
| --- | --- |
| `build-image <target> --tag <tag> [--platform <p>] [--push] [--no-record]` | Build the per-target Docker image. `--push` records the registry manifest digest in `docker/digests.json`; `--no-record` opts out. |
| `cloud deploy --provider <p> --region <r> [--tier <t>] [--image-tag <tag>]` | Deploy a managed CrewHaus cluster to AWS / GCP / Azure / LocalStack. |
| `cloud teardown --provider <p> --region <r>` | Tear down a managed cluster. |
| `channel provision <spec> --base-url <public-url> [--platform slack\|telegram\|discord\|all] [-o <dir>] [--dry-run] [--force]` | One-command platform app setup for a channel spec: Slack app manifest YAML (including the reaction scopes the ratings feature needs), Telegram `setWebhook`, Discord interactions endpoint + invite URL. |
| `channel verify <spec> [--platform …] [--base-url <url>] [--dry-run]` | Scope doctor: Slack `auth.test` + granted scopes, Telegram `getWebhookInfo`, Discord application fetch (exit 1 on missing scopes / mismatched webhook). |
| `federation discover <deployment> [--srv-domain <d>] [--format json\|yaml]` | Resolve a federated peer's endpoint + cert fingerprint via DNS SRV or `.well-known`. |
| `sandbox doctor [--probe] [--format json\|table]` | List registered sandbox images + healthcheck status. |

---

## See also

- [GETTING-STARTED.md](GETTING-STARTED.md) — the guided tour, the spec
  reference (including the v0.2.0 spec blocks), the runtime directory, and
  the permission model.
- [HANGAR.md](HANGAR.md) — the harness manager in full: the console, the
  registry, the supervisor's on-disk contract, `crewhaus.control.v1`, the
  security model, and troubleshooting.
- [PROVIDERS.md](PROVIDERS.md) — the model-string grammar every `--model`
  flag and `model:` field accepts.
- [COMPILER-ARCHITECTURE.md](COMPILER-ARCHITECTURE.md) — what `compile`
  actually does between YAML and TypeScript.
- The [demos](https://github.com/crewhaus/demos) repo's walkthroughs — one
  per capability, each exercising the commands above end to end.
