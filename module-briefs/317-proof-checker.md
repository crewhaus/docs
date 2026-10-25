# proof-checker

Status: implemented and tested
Dependency phase: 9 - Telemetry & eval
Catalog layer: R15 - Telemetry, Tracing, Eval
Origin in ordering: 0.8.0 — the provability release (design §6.4)
Workspace home: packages/proof-checker
Targets: CLI, CHN, MGD (behind `ProofCheck`, `ProofCheckers` and the `proven` rung; `crewhaus provability checkers` and `crewhaus doctor --probe` probe the same checkers)
Test layers: T1, T2, T8

## Purpose

The pluggable proof checkers behind the provability ladder's `proven` rung. A checker **checks**; the model searches. `ProofCheck` (in `tool-proof`) hands a checker the model's statement and proof, and the checker's verdict is the only thing the `proven` rung trusts. No model call anywhere in the package.

Three ship behind one interface. `builtin` makes `proven` reachable with no sandbox at all, because most theorem-shaped claims a harness makes are finite or arithmetic; `smt` covers the arithmetic middle; `lean4` is there for the rest.

## Boundaries

Owns:

- **The contract** (`types.ts`) — `ProofChecker { id, language, available(), check(input) }`. `language` is one paragraph written for the model, which `ProofCheckers` shows verbatim. `check()` never throws: every refusal is `{ ok: false, reason }` in one sentence, with a `counterexample` when the checker found one — the retry nudge's best input. A checker of your own implements the same interface and reaches the tools through `createCheckerRegistry` or `tool-proof`'s `checkers` option; a spec cannot name one yet, since `formal.checkers` takes `builtin`, `smt` and `lean4` and the plugin loader has no slot for a checker (design §12).
- **`builtin`** (`builtin/`) — a dependency-free certificate checker, always available. Statements use exact bigint rationals (never floats), comparisons over arithmetic terms with variables, propositional connectives, `let` and arrays, and bounded quantifiers over explicit finite domains. A proof is a JSON list of steps, each justified by one of `given`, `eval`, `arith`, `subst`, `mp`, `and_intro`, `and_elim`, `or_intro`, `cases`, `instantiate` or `witness` — or the single word `eval`, for a closed statement true by evaluation. `contra` exists only to reject: reaching φ and `not φ` shows the hypotheses inconsistent. Givens are recorded as assumptions; inconsistent or never-true hypotheses are rejected, so a closed statement is proven only when it is true. No induction and no unbounded quantification — the refusal names `smt` and `lean4`. Nothing is executed: the checker parses text and evaluates exact arithmetic.
- **The builtin's bounds** — parsing, every rule and every loop spend from one budget (`BUILTIN_LIMITS`), with a wall-clock deadline capped whatever the caller passes, so a spent budget is a rejection, never a hang. Each check runs on a worker thread of its own, so the harness's event loop — every session, every tenant of a managed daemon — keeps running while it does. The worker's source is generated into the package as text (`scripts/gen-worker-source.ts`, with a test that fails until it is regenerated) so a `bun build --compile` binary carries it.
- **`smt`** (`smt.ts`) — SMT-LIB 2 checked by Z3 in a sandbox. `statement` is the goal as one term; `proof` is a preamble limited to declarations, definitions and `assert`, and anything else is refused before the container runs, as is any name that would redefine one of Z3's built-ins (`SMT_RESERVED_NAMES`) — a redefined `<=` would "prove" anything. The adapter writes the script itself and runs it once: the preamble must be satisfiable, then the negated goal is asserted — `unsat` ⇒ proven, `sat` ⇒ rejected with Z3's model as the counterexample, `unknown` or a timeout ⇒ rejected. Every `assert` and `define-fun` is reported as an assumption, since a definition can fix the very value the goal states.
- **`lean4`** (`lean4.ts`) — Lean 4 in a sandbox. The adapter puts the statement and proof inside one `theorem`, followed by its own `#print axioms`, after a deterministic pre-scan over exactly that text with comments and literals blanked. The scan refuses the `formal.forbid` tokens (default `sorry`, `axiom`, `native_decide`); any command keyword, `#command` or attribute that could end the theorem and start another; imports outside `Init` and `Std` (plus `Mathlib` when `formal.image` overrides the curated image); and a statement that could end the theorem's type early. `sorryAx` is always a rejection, and so is an axiom through which a proof trusts the compiler while `native_decide` is forbidden. Any other axiom beyond `propext`, `Classical.choice` and `Quot.sound` is recorded and shown, not hidden.
- **The sandbox posture** (`sandbox-common.ts`) — `smt` and `lean4` run only in a container sandbox (`@crewhaus/sandbox`'s defaults: no network, read-only root, a `/tmp` tmpfs, memory and CPU caps) and refuse on the `noop` backend, which would run the checker on the host, in both `available()` and `check()`. The refusal lives in the checker because `ProofCheck` is deliberately not `requiresSandbox`: the builtin needs none. `available()` runs the image's healthcheck once, remembers a pass, and remembers a failure only briefly (`probeRetryMs`), so a `ProofCheckers` call does not re-probe every time.
- **The registry** (`registry.ts`) — `createDefaultCheckers` turns `formal.*` into checkers in `formal.checkers` order, the preference told to the model. With no sandbox, `smt` and `lean4` are still listed and say why they cannot run, so the model sees what it could prove with instead of a checker silently missing. `createCheckerRegistry` refuses a repeated id, and `checkerImages` lists the images a sandbox must allow.

Does not own: persisting a verdict as an artifact, publishing `proof_checked`, or re-checking a stored proof (`tool-proof`); whether an artifact backs a sentence (`provability`'s `proven` verifier); the images themselves (`sandbox-image-z3`, `sandbox-image-lean4` — this package re-exports their pinned refs as `DEFAULT_SMT_IMAGE` / `DEFAULT_LEAN4_IMAGE`); and the container runtime (`sandbox`).

## Inputs and Outputs

Inputs: `{ claim, statement, proof, timeoutMs?, signal? }` per check; at construction, the `formal` settings (checker ids, `forbid`, `timeoutMs`, the lean4 image override) and a sandbox for the sandbox checkers.

Outputs: `CheckResult { ok, checker, assumptions, axioms, output, counterexample?, reason?, durationMs }`, and `CheckerAvailability { ok, reason? }`.

## Dependency Notes

Depends on `@crewhaus/sandbox`, `@crewhaus/sandbox-image-z3`, `@crewhaus/sandbox-image-lean4` and `@crewhaus/errors`. Consumers: `tool-proof` (which builds the checkers for a harness and creates its sandbox lazily), `evaluation-service` (the sandbox type) and `apps/cli`.

`formal.timeout_ms` (sized for a container's cold start) reaches the sandbox checkers whole, but never stretches a builtin check past its own cap: a builtin check holds a core for its whole budget.

## First Implementation Slice

Built in the release's first stage: the interface, the `builtin` checker (so the ladder is usable with no external image), the `smt` and `lean4` adapters with the lean pre-scan, and the registry. The images and the live tests came with the third stage, and the adapters were then held to what the real toolchains print. Moving each builtin check onto a worker thread came with the first whole-release review, which found a synchronous check stalling the event loop.

## Study References

Z3's SMT-LIB 2 interface and `(get-model)`, Lean 4's `#print axioms` and its JSON message output, the curated-image registrations in `sandbox-image-*`, `tool-code-execution`'s sandbox floor (the posture the sandbox checkers take), and design §6.4 and §10.

## Validation Plan

Catalog tests: T1, T2, T8. Primary risks: **an unsound acceptance** — a rule, a redefined built-in or a spliced Lean file that proves what is not true — and **proof text as an attack**, since it is model output that may have absorbed injected content and the sandbox checkers hand it to a program. Every builtin rule is tested accepting and rejecting; the smt and lean4 parsers are held to output captured verbatim from the curated images' Z3 and Lean, so they run without Docker; and a live suite (`CREWHAUS_SECTION36_LIVE_DOCKER=1`, after `bun scripts/build-sandbox-images.ts`) runs both checkers on the real images as a harness does and re-probes the image's Z3 for built-in names the reserved list lacks.

Residual, documented: open givens whose inconsistency cannot be read off the formulas are trusted, and listed as assumptions on every surface.

Definition of done: `bun test packages/proof-checker/src` green, the live suite green on freshly built images, and design §0's scenario C — the six-meetings claim proven, its false variant rejected with the overlapping pair as the counterexample.
