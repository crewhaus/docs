# tool-proof

Status: implemented and tested
Dependency phase: 11 - Built-in tool implementations
Catalog layer: R4 - Built-in Tool Implementations
Origin in ordering: 0.8.0 — the provability release (design §7)
Workspace home: packages/tool-proof
Targets: CLI, CHN, MGD (bound to the gate by `evaluation-service`); a spec that lists the tools in `tools:` without a `provability:` block gets the unbound builtin instances
Test layers: T1, T3, T8

## Purpose

The tool surface of the provability ladder: the tools a harness proves its claims with, and checks a draft with before it answers. Category `proof` (`all-proof`), rolled up under `safety` beside `verify`, so `all-safety` grants both.

| Tool | What it does |
|---|---|
| `ProofCheckers` | Lists the proof checkers this harness can use, in preference order: whether each can run here (and why not), and the statement language each one checks. The model should never guess what it can prove with. |
| `ProofCheck` | Runs a checker on `{ checker, claim, statement, proof }` and saves the verdict as a `prf_…` artifact, accepted or rejected. On ok the answer cites it as `[n]: proof:prf_…`. |
| `ProvabilityPreflight` | Grades `{ draft }` with the gate's own verifier over the session's evidence and returns the verdict, a row per claim and what to fix. Gates nothing. |

Nothing here cites for the model, and nothing calls a model. Sources ground on their own from the tool calls the model already makes; tools appear only where there is real work to do (`ProofCheck`) or a question to answer.

## Boundaries

Owns:

- **The three tools and their flags.** `ProofCheckers` and `ProvabilityPreflight` are read-only and concurrency-safe. `ProofCheck` is destructive (it writes one artifact) and concurrency-safe, and deliberately not `requiresSandbox`: the builtin checker needs no sandbox, so a tool-level flag would deny it on every sandbox-less harness. The sandbox floor is per checker instead — `smt` and `lean4` refuse themselves without a container sandbox and on the `noop` backend.
- **The operative argument** — `ProofCheck` declares `checker`, so a permission rule can scope it. permission-engine carries `alwaysAllow ProofCheck(builtin)` as a builtin default, beside `ProvabilityPreflight` and `ProofCheckers`, so a `proven` harness reaches its floor headless and the gate's own nudge never parks a turn; `ProofCheck(smt)` and `ProofCheck(lean4)` start a container and keep the mode default.
- **Artifacts and the event** — every check is saved, accepted or rejected, under the proofs root as `artifacts/<prf_id>.json` (mode 0600, never overwritten). A rejected proof is kept because proof pressure is signal — the `action_proof` rule from tool-plan — and the bundle can then say why a citation of it failed. A check that never finishes is saved as a rejection once it runs past `formal.timeoutMs` plus a grace period; a cancelled call saves nothing. Each check publishes `proof_checked` (checker, ok, artifact id, duration, axioms) on the run's bus and appends the same line to the session event log.
- **`createProofTools(opts)`** — binds the three tools to one configuration: the lowered `provability:` block, the spec name and the evidence builder. `evaluation-service` calls it with the gate's own, so the tools and the gate agree on the floor, the checkers, the artifacts root and the evidence. Roots are resolved on every call for the tenant the call runs under, so one instance built at a managed daemon's boot serves every tenant; a root outside the call's tenant is refused.
- **The builtin instances** — what a spec gets by listing the tools in `tools:` with no `provability:` block: the compiler's defaults at the `grounded` floor, checkers in `builtin, smt` order, artifacts under `.crewhaus/proofs/default/`, and the run's own session log as evidence.
- **The preflight's three differences from the gate.** It never leaves the run: no HEAD request (the draft is checked as if `sources.resolve` were `off`), no replay (bindings read `replayed: false`), and no waiver judge, each with a note saying so. On a paraphrase the gate's judge might accept, the preflight is the stricter of the two.
- **`recheckProof`** — the `proven` rung's second reading of a stored artifact: the builtin checker run again, bounded by `formal.timeoutMs`, on the statement and proof the file holds now. A sandbox checker's artifact rests on the logged call alone, since re-running it would start a container per check.
- **Reports sized for the model** — the verdict and the next step first, then the details, small enough to read whole because the runtime shows only the start of a large result; the full proof and output stay in the artifact.

Does not own: the checkers (`proof-checker`); the verifier the preflight runs (`provability`'s `verifyDocument`, the function the gate calls); whether a `proof:` citation backs a sentence (`provability`'s `proven` verifier); and putting the bound tools on a run (`evaluation-service`'s `registerGateTools` / `mergeGateTools`, which replace a same-named builtin instance with its bound twin).

## Inputs and Outputs

Inputs: the model's tool calls; at construction, the provability config, the spec name, the workspace root, the proofs and session roots (or functions of the tenant), an evidence builder, and optionally a prebuilt checker list (`checkers`); without one the tools build the default checkers and create their sandbox lazily. `evaluation-service` passes a list built over its own sandbox when its caller gives it one.

Outputs: compact tool reports, persisted proof artifacts, and `proof_checked` events on the bus and in the session log.

## Dependency Notes

Depends on `@crewhaus/proof-checker`, `@crewhaus/provability`, `@crewhaus/sandbox`, `@crewhaus/tenancy`, `@crewhaus/tool-builder`, `@crewhaus/tool-catalog`, `@crewhaus/tool-categories`, `@crewhaus/event-log`, `@crewhaus/trace-event-bus`, `@crewhaus/errors` and `zod`. The sandbox is created lazily, on the first call that needs a sandbox checker, the way `tool-code-execution` creates its own: importing the package (the CLI imports every tool package) never touches a container runtime.

Registered like every builtin: three entries in tool-categories' `BUILTIN_TOOLS` and the CLI's tool-package map, the places `scripts/wire-tool-package.ts` writes.

## First Implementation Slice

Built in the release's second stage, beside the gate it binds to: the three tools, artifact persistence and the `proof_checked` event, wired by the manifest script, with tenant-fenced roots and an evidence builder that applies `scope: turn` the gate's way added by that stage's review. The re-check (`recheckProof`) came with the first whole-release review, which exercised forged artifact files, and `ProofCheck(builtin)` joined the builtin allow rules in the review after it.

## Study References

`tool-plan` (the `action_proof` rule: a rejected attempt is evidence too), `ListTools` / `ToolRegistry` (the "never guess what you can use" precedent behind `ProofCheckers`), `tool-code-execution` (lazy sandbox creation), and design §6.4 and §7.

## Validation Plan

Catalog tests: T1, T3, T8. Primary risks: **a tool and a gate that disagree** — answered by one configuration through `createProofTools` and a preflight that is `verifyDocument` itself; **a forged artifact** — `provability` requires the run's own logged `ProofCheck` call, and the builtin re-check reads the file again; **a headless turn parked on an approval** — the builtin allow rules; and **a cross-tenant read or write** — per-call roots and the refusal of a root outside the tenant, covered by `tenancy.test.ts`.

Definition of done: `bun test packages/tool-proof/src` green, and a `proven` harness under default permissions proving and citing a claim through the builtin checker with no approval prompt.
