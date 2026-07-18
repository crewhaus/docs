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
from the current working directory). Install it however you like; see
[GETTING-STARTED](GETTING-STARTED.md) for the install matrix.

> **Everything here is additive and opt-in.** As of v0.2.0 the CLI grew from
> a handful of build/run/eval verbs into a full lifecycle surface — but
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

## Table of contents

1. [Build, run, and author](#build-run-and-author)
2. [Health and diagnostics — `doctor`](#health-and-diagnostics--doctor)
3. [The self-building eval flywheel](#the-self-building-eval-flywheel)
4. [Datasets, graders, and judges](#datasets-graders-and-judges)
5. [The observer/advisor](#the-observeradvisor)
6. [Model and cost automation](#model-and-cost-automation)
7. [Feedback, memory, and knowledge](#feedback-memory-and-knowledge)
8. [Self-healing operations](#self-healing-operations)
9. [Deploy, promote, and govern](#deploy-promote-and-govern)
10. [Fleet, lifecycle, and marketplace](#fleet-lifecycle-and-marketplace)
11. [Safety that learns](#safety-that-learns)
12. [Compliance, audit, and retention](#compliance-audit-and-retention)
13. [Registry, secrets, and state](#registry-secrets-and-state)
14. [Distribution and infrastructure](#distribution-and-infrastructure)

Flag conventions in this document: `<required>` positional or value,
`[--optional]`, `a|b|c` an enum of choices, `-o` / `--out` a short alias.
Values marked `registry:<ref>` accept the
`registry:<name>[@version][#split]` shorthand for a dataset stored in the
local registry (see [`datasets`](#datasets-graders-and-judges)).

---

## Build, run, and author

The everyday loop: scaffold a spec, compile it to a bundle, run it.

| Command | Purpose |
| --- | --- |
| `init [name]` | Scaffold a fresh `crewhaus.yaml`. |
| `init --interactive [--detect]` | Interview-driven spec authoring (via the resolved model, or a scripted stdin questionnaire with no credentials); the result is validated with `parseSpec`. |
| `init --ci` | Also scaffold `.github/workflows/crewhaus-eval.yml` — an eval-gated spec-PR check. Composable: run it in an existing harness to add just the workflow. |
| `init --with-evals` | Also scaffold `eval/dataset.jsonl` + `eval/graders.yaml` (offline template mode — no credentials needed) so the harness can `crewhaus eval` on day one. |
| `init --sentinel` | Also scaffold `.github/workflows/sentinel-drift.yml`, the nightly model-drift sentinel cron. |
| `compile <spec> -o <out-dir>` | Parse → IR → emit a runnable bundle. **The FR-002 external-sink scope gate runs by default** — the build fails if an I/O-capable tool is left at a non-`external` scope. |
| `compile <spec> --emit-ir [-o <dir>]` | Print the lowered IR as JSON (or write `<dir>/ir.json`); skip codegen. |
| `compile <spec> --emit-loop [--json] [-o <dir>]` | Skip codegen and print the canonical **agent-loop projection** (`projectLoop` of the lowered IR) — the exact wire shape the Studio `/builder` renders and the compiler-worker's `POST /loop` returns. Human-readable by default; `--json` prints the raw `LoopProjection`; `-o` writes `<dir>/loop.json`. A read-only view: nothing is emitted and (matching `POST /loop`) the FR-002 scope gate does **not** run, so you can inspect the loop of a spec whose tool scopes still need fixing. Mutually exclusive with `--emit-ir`. |
| `compile <spec> --emit-as local\|cf-worker` | `local` (default) emits the standalone Bun bundle; `cf-worker` emits the Cloudflare-Worker bundle (`worker.js` + `wrangler.toml` + `package.json`) — the **same** bundle the Studio's remote compiler (`compiler-worker POST /compile`) serves. Supported for `target: cli\|workflow\|graph`; other shapes are rejected. Incompatible with `--emit-ir`/`--emit-loop`/`--check`/`--with-eval-harness`. |
| `compile <spec> --strict` | Escalate compile **warnings** (accepted-but-unwired spec keys) to errors — warnings always print (code + path + message, one per line), and with `--strict` any warning fails the compile before files are written. Distinct from the default-on FR-002 external-sink scope gate (which is governed solely by `--allow-unmarked-sinks`). |
| `compile <spec> --check` | After emitting, verify the bundle: shape smoke assertion → `bun install` → credential-free liveness boot. Red exits 1. |
| `compile <spec> --watch` | Re-run parse → lint → compile on every change to the spec / commands / skills dirs. Debounced, Ctrl-C-clean, one green/red status per cycle. |
| `compile <spec> --with-eval-harness [--eval-dataset <name>]` | Also emit an eval bridge (a `target: eval` bundle projected from a non-cli shape's own agent) into `<out-dir>/eval/`, so that shape can consume its distilled feedback through eval/optimize/flywheel. |
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
| `serve --mcp <spec> [--sse] [--port <n>] [--plugins <a,b>]` | Project the spec's cli agent as an **MCP server** so it becomes a tool inside Claude Code / an IDE / another CrewHaus runtime. `--mcp` is the only projection kind today. Default transport is stdio; `--sse` serves over HTTP+SSE (a `fetch` server, default port 8000 or `CREWHAUS_MCP_PORT`), overriding the spec's `expose.mcp.transport`. |
| `runs resume <session> [--spec <path>] [--prompt <text>]` | Re-drive a persisted cli session — e.g. one PARKED on a pending approval (resolved via `approvals grant`). Resolves the backing spec from `--spec` or `cwd/crewhaus.yaml`; the run flags a resumed continuation honours (`--model`, `--budget-usd`, `--trace`, `--streaming`, `--justification-judge`, `--egress-matcher`, `--user`, …) are threaded verbatim. `runs` is the run-lifecycle namespace; `resume` is its only CLI verb today. |
| `approvals list\|show\|grant\|deny <id> [--dir <root>] [--by <who>] [--json]` | Resolve tool-permission approvals a headless run parked under `permissions.ask_mode: pause`: `list`/`show` the pending queue, `grant`/`deny <id>` a decision (recorded with the deciding identity from `--by` / `CREWHAUS_USER`, default `cli`). `grant --once` applies to the next matching tool call only (the default; the runtime consumes a grant on use). After granting, resume the parked run with `runs resume <session>`. |
| `export claude-plugin <spec> [--out <dir>] [--force] [--author <name>] [--author-email <e>] [--description <d>]` | Emit an Anthropic-compatible **Claude Code plugin** directory from any target shape. Output defaults to `<cwd>/<pluginName>` and refuses to overwrite a non-empty dir without `--force`. `--author` (default `CrewHaus`) is stamped into `.claude-plugin/plugin.json` (Anthropic's schema requires a non-empty author). |
| `upgrade [spec] [--write]` | Detect the spec's `version:` drift vs the current CLI and run the migration chain (validated). Dry-run diff by default; `--write` applies. |
| `migrate-all --from <N> --to <N> [--dry-run]` | Batch-migrate every spec in the registry to a newer IR version. |
| `context --bundle [-o <file>]` | Emit a single-markdown orientation manifest (`--factory-root` / `--docs-root` / `--demos-root` locate the sources). |

`init` scaffold flags (`--ci`, `--with-evals`, `--sentinel`) are composable
with each other and with an existing harness; `--force` overwrites a
previously scaffolded workflow or eval assets (never the spec).

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

| Command | Purpose |
| --- | --- |
| `flywheel run [spec]` | The whole loop, one command: compile gate → baseline eval → optimize → after eval → acceptance gate (pass rate strictly up **and** zero regressions) → write-back on accept. A rejected patch never touches the spec. Flags: `--dataset`, `--graders`, `--budget-usd`, `--iterations`, `--seed`, `--concurrency`, `--mutator rule-based\|claude`, `--dry-run` (rehearse, never write), `--allow-dirty` (opt out of the clean-tree invariant). |
| `flywheel init [--force]` | Scaffold `.github/workflows/crewhaus-flywheel.yml` (nightly cron + manual dispatch). Accepted improvements arrive as PRs for human review — never auto-merged. |
| `eval <spec> --dataset <d> --graders <g>` | Run the agent against a dataset and grade it (deterministic graders + LLM-as-judge). `--dataset` also accepts `registry:<ref>`. Flags: `--judge-model`, `--concurrency`, `--seed`, `--repeats`, `-o`. |
| `eval … --repeats <K>` | Run every sample K times (trial *t* gets `seed+t-1` when `--seed` is set; i.i.d. draws otherwise). Trial 1 is the canonical result; per-trial grades land on the sample (`results.json` `trials[]`) and the aggregates gain **pass@K** (at least one trial passed — the optimistic capability metric) and **pass^K** (all K passed — tau-bench's reliability metric: a flaky 60%-reliable agent scores 0.6^K). Trials run sequentially inside each sample's concurrency slot, so a K-repeat run costs ~K× the wall clock and spend (`tokens_all_trials` makes the real spend visible). |
| `eval … --gate` | Exit non-zero on a regression vs the pinned `(spec, dataset)` baseline (strict: any pass-rate drop or pass→fail flip). |
| `eval … --no-promote` | Keep the existing baseline pin instead of auto-promoting this run on a gate pass. |
| `eval … --models <m1,m2,…>` | Benchmark matrix: run the same dataset+graders once per model; each cell writes to `<out>/<model-slug>/` and the run emits `matrix.json` + `index.html`. Incompatible with `--gate`/`--no-promote`. |
| `eval … --no-regressions` | Skip the default union of the per-spec `<name>-regressions` registry dataset (also skips the failure-arbiter's bug-sample pin into it). |
| `eval … --no-retry` | Opt out of the runner's default one-shot retry of ERRORED samples (infra noise, not graded failures). |
| `eval … --sentinel --baseline <run-dir>` | Nightly model-drift sentinel: re-run the seed-pinned dataset against the unchanged spec and diff against a frozen baseline; a flip/score-shift when specHash **and** dataset-hash are both unchanged is provider drift → exit non-zero. |
| `eval … --voice [--replay-dir <d>] [--max-ttft-ms N] [--max-turn-latency-ms N] [--max-barge-in-yield-ms N]` | Voice replay eval: replay recorded call-session logs through the voice grader pack (latency / barge-in / transcript). |
| `eval coverage [--sessions N\|all] [--dataset <d>] [-o <dir>] [--format text\|html\|json]` | Detect production behaviors (tool / MCP / bigram / compaction patterns) that no eval sample exercises, ranked by production frequency. The `json` form is a backlog for `dataset mine`. |
| `eval-report diff <prevRun> <newRun> [-o <dir>]` | Compare two eval runs and emit a diff report highlighting pass/fail flips. |
| `eval-report history [--spec <n>] [--dataset <n>]` | List recorded runs from `.crewhaus/evals/index.jsonl`. |
| `eval-report baseline show [--spec <n>] [--dataset <n>]` | Print pinned baselines from `.crewhaus/evals/baselines.json`. |
| `eval-report baseline set <runId>` | Pin a recorded run as its `(spec, dataset)` baseline. |
| `optimize <spec> --dataset <d> --graders <g>` | Active eval-driven optimization. `--dataset` accepts `registry:<ref>`. Flags: `--mutator rule-based\|claude`, `--iterations`, `--seed`, `--concurrency`, `--improvement-threshold`, `--budget-usd` (stop a model-driven run before it exceeds $N), `--write-back`, `-o`. |
| `optimize … --ratings <session>\|all` | Inline-distill user ratings into the training set (synthesizes the dataset, and the graders when `--graders` is omitted). |
| `optimize … --from-advice <suggestions.json>` | Apply `advise` SpecPatches through the eval-gated accept/reject/compose loop instead of running the mutation search. |
| `optimize … --few-shot <pool>\|auto [--few-shot-k N]` | Inject the top-K harvested few-shot examples at the front of the seed instructions the optimizer mutates. |
| `optimize … --no-pin-regressions` | Skip pinning an accepted patch's fail→pass recoveries into `<name>-regressions`. |
| `scaffold-evals <spec> [-o <dir>] [--samples N] [--model <m>] [--force]` | Day-one eval assets **from the spec**: sample stubs derived from `agent.instructions` (one model call with credentials, deterministic template without) + one starter grader (a spec-goal `llm_judge` rubric online, a non-empty-answer floor grader offline). |

---

## Datasets, graders, and judges

The versioned dataset registry, grader drafting, dataset growth from
production, and judge calibration.

| Command | Purpose |
| --- | --- |
| `datasets list` | All registered datasets + versions. |
| `datasets get <name>[@version] [--split train\|dev\|test]` | Print a dataset's samples as JSONL (one split, or the all-splits merge). |
| `datasets put <name> --file <f.jsonl> [--split-spec 70/15/15 \| --split <name>]` | Import a file as a new auto-bumped version. |
| `dataset mine [--sessions N\|all] [--out-dataset <name>] [--review [--yes]]` | Mine hard cases from session struggle signals (tool-errors, loops, retries, egress blocks) into a quarantine dataset; `--review` promotes accepted candidates into a mined registry dataset (interactive in a TTY; `--yes` required to accept non-interactively). |
| `dataset synthesize --from <file\|registry:ref> [--count N] [--budget-usd N] [--out-dataset <name>] [--model <m>]` | PII-redacted stress variants (paraphrase / truncate / ambiguate / inject) into a separate synthetic split that never contaminates human golds. |
| `dataset refresh-goldens --dataset <file\|registry:ref> [--min-score F] [--apply] [--model <m>]` | Reconcile user corrections and up-rated turns with existing golds; propose gold updates as a review diff. `--apply` writes a **new** registry version — never in-place. |
| `distill --session <id> -o <ds.jsonl>` | Turn ratings on a session's turns into an eval dataset + one grader. Flags: `--all-sessions`, `--graders-out <g.yaml>`, `--min-score`, `--judge [--judge-model <m>]` (emit an `llm_judge` grader instead), `--register <name>` (also promote a new dataset version into the registry). |
| `graders suggest [-o <file>] [--runs <dir>\|last:N] [--model <m>] [--spec <n>] [--min-score F] [--force]` | Draft grader suites from failure rationale: cluster `grades.json` rationale (via the run-history index), judge criterion scores, and rating comments into themes; draft deterministic graders per theme (+ an `llm_judge` rubric with `--model`) into a **review file** — never auto-applied. |
| `judge calibrate [--graders <g.yaml>] [--dataset <d>] [--model <m>] [--sessions N\|all] [--apply]` | Calibrate an `llm_judge` against human ratings: correlation / bias / ROC-optimal cut point over paired `(human rating, judge score)` samples. `--apply` writes the calibrated `--min-score` default. |

---

## The observer/advisor

Suggestions that reach beyond the prompt — mined from durable session
telemetry, validated against the eval gate before anything is applied.

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
bundles. See the [spec reference](GETTING-STARTED.md#model-cost-and-budget-blocks-v020).

---

## Feedback, memory, and knowledge

Capture ratings, harvest golden examples, and let a harness get smarter from
its own history.

| Command | Purpose |
| --- | --- |
| `rate --session <id> [--turn N] (--thumbs up\|down \| --stars 1-5 \| --score 0-1) [--comment <t>] [--rater <id>]` | Rate an assistant turn. |
| `feedback --session <id> --text <msg> [--turn N] [--correction <better answer>] [--rater <id>]` | Attach a comment or a correction (a better answer) to a turn. |
| `fewshot harvest [--all-sessions] [--min-score F] [-o <pool>]` | Harvest up-rated turns into a golden few-shot pool (PII/secret-redacted); consumed by `optimize --few-shot`. |
| `fewshot show [--k N]` | Print the pool as the injectable prompt block. |
| `faq distill [--sessions N\|all] [--min-score F] [--min-occurrences N] [-o <skill-dir>]` | Cluster recurring user questions into an auto-discovered FAQ skill under `.crewhaus/skills/faq/`. |
| `lessons update [--sessions N\|all] [--low-score F] [-o <LESSONS.md>]` | Mine corrections + failure→fix patterns into a deduped, auto-loaded `LESSONS.md`, plus per-user preference files under `.crewhaus/preferences/`. |
| `sessions summarize [--before <date>] [--evicted [--ttl-days N]]` | Fold sessions into a durable index; `--evicted` indexes each session just before TTL eviction deletes it (the summarize-before-evict hook). |
| `sessions tail [<session>] [--dir <root>] [--no-follow] [--interval <ms>]` | Follow a session's transcript live — a `tail -f` for a running agent (the per-turn view `crewhaus dev` points at). With no `<session>`, tails the most-recently-updated one under `.crewhaus/sessions`; each user/assistant turn, tool call + result, and failure prints one line as it lands. `--no-follow` dumps the current transcript and exits (scriptable/CI); `--interval` sets the poll ms (default 500). |
| `sessions export --format trajectories [--out <file.jsonl>]` | Emit one JSONL line per agent step — a `(state, action, observation, reward)` tuple — from every session event log under `.crewhaus/sessions`. `reward` is terminal-sparse (null except the last step, which carries the last `eval_graded` score, else the latest user rating normalized to `[0,1]`, else null; `rewardSource` says which rung fired). `--out` writes to a file; omitted, it streams to stdout. **Experimental (G53):** inference-time scaffolding (eval → optimize → flywheel) stays the mature improvement lane — this export exists so an external trainer can consume real sessions, and is deliberately **not** wired into `crewhaus optimize`. |
| `knowledge sync [--pull \| --push] [--root <dir>] [--shared <dir>] [--dry-run] [--no-redact]` | Cross-harness knowledge sync: publish/import shared memories, distilled grader suites, and optimizer-winning instruction fragments to a versioned shared store. Redacted by default. |

Auto-capture/recall is also a declarative spec block — `memory:` wires the
Remember/Recall tools and the capture/recall passes; `feedback:` configures
the capture surfaces and `autoDistill`/`exitPrompt`. See the
[spec reference](GETTING-STARTED.md#memory-and-feedback-blocks-v020).

---

## Self-healing operations

Canary deploys, drift sentinels, MCP health, incident bundles, and load
testing — the operational safety net for a running harness.

| Command | Purpose |
| --- | --- |
| `deploy canary <spec> <version> --traffic 5,25,50,100 --dataset <d> --graders <g>` | Eval-gated ramp with auto-rollback: register a candidate, step traffic, eval both versions per step, gate on the regression runner, auto-promote or auto-rollback — all audit-logged. Also: `--env`, `--name`, `--from`, `--concurrency`, `--seed`, `--judge-model`, `--max-pass-rate-drop`, `--max-p95-latency-ms`. |
| `mcp doctor [--probe] [--sessions N] [--format json\|table]` | Per-server MCP health scoring from `mcp_call` events, tool-schema drift watch (`--probe` does a live `listTools`), and the runtime auto-quarantine decision. |
| `incident collect --session <id> [--kind <k>] [--reason <r>] [-o <dir>]` | Assemble an incident bundle from a session's traces + audit + cost + `doctor` output. |
| `failures report [--sessions N\|all] [--propose-taxonomy] [--dir <root>] [-o <file>] [--json]` | Aggregate and cluster `run_failed` events + incident records by failure class and message similarity across recent sessions. `--propose-taxonomy` drafts `failure_taxonomy` spec entries from the clusters (reusing the advise drafting machinery + specificity floor) to stdout or `-o`. |
| `loadtest <spec> [-c N] [-n N] [--duration <d>] [--rps N] [--gate] [--max-p95-ms N] [--max-error-rate F] [--stub-latency-ms N] [--format <f>] [-o <dir>]` | Replay a dataset against a locally-booted daemon at N concurrent sessions: p50/p95, TTFT, breaker/rate-limit trips, cost per request. `--gate` exits 1 when p95 latency / error rate exceed the thresholds. |
| `intents [--sessions N\|all] [--top N] [--format <f>] [-o <dir>]` | End-user intent analytics digest: clustered top intents, low-rated / no-answer clusters, tool-failure-correlated requests, week-over-week trends. |

The `observability.slo:` spec block declares production SLOs and the
mitigation ladder the runtime walks on a sustained breach; see the
[spec reference](GETTING-STARTED.md#observability-and-slo-block-v020).

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
| `propose <proposed.yaml> [--current <spec>] [--source optimize\|advise\|model-scan\|manual] [--as-version <v>] [--dry-run]` | Package a spec change into a review artifact and open a PR (the governance wrapper around optimize/advise write-back). Never auto-merges. Also folds in optimize provenance via `--optimize-dir`, `--run-id`, `--score-before`/`--score-after`. |

`compile` and `optimize --write-back` auto-register the changed spec into
the local registry with a distilled changelog; `compile --no-register` /
`optimize --no-register` opt out.

---

## Fleet, lifecycle, and marketplace

Operate more than one harness, share what they learn, and decommission
cleanly.

| Command | Purpose |
| --- | --- |
| `fleet list [--root <dir>]` | Cross-harness inventory (any directory carrying a `crewhaus.yaml`; discovery skips `.crewhaus/`, `node_modules/`, `.git/`, `dist/`). |
| `fleet status [--root <dir>]` | Per-harness health rollup. |
| `fleet run <sub> [--filter <glob>] [--root <dir>] [--allow-mutating] [--yes]` | Bulk-run a subcommand across the filtered fleet. Read-only bulk subcommands (`eval`, `doctor`, `security digest`, `audit verify`) run freely; a mutating subcommand requires `--allow-mutating` and per-harness confirmation. |
| `retire <spec> [--archive <dir>] [--dry-run] [--force] [--push-knowledge [--shared <dir>]]` | Audited harness decommissioning: archive sessions/feedback/memories, retention-purge, rotate-then-revoke secrets, tombstone registry entries, emit a final compliance bundle. `--force` retires despite an active pin; `--dry-run` prints the plan. |
| `plugins list\|search\|install\|uninstall\|outdated\|publish …` | Marketplace plugins CLI. `search -q <text>`, `install <name> [--version <v>]`, `publish --manifest <plugin.json>` (opens a publish PR). Flags: `--registry <dir\|url>`, `--plugins-dir`, `--allow-unsigned` (dev only), `--dry-run`. |
| `templates list\|search\|use …` | Marketplace templates CLI. `search -q <text> [--target <t>]`, `use <name> [--into <dir>] [--subdir <s>]` scaffolds a template into a workspace. |

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
- [PROVIDERS.md](PROVIDERS.md) — the model-string grammar every `--model`
  flag and `model:` field accepts.
- [COMPILER-ARCHITECTURE.md](COMPILER-ARCHITECTURE.md) — what `compile`
  actually does between YAML and TypeScript.
- The [demos](https://github.com/crewhaus/demos) repo's walkthroughs — one
  per capability, each exercising the commands above end to end.
