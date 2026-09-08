# preflight

Status: implemented and tested
Dependency phase: 18 - Deployment & studio
Catalog layer: F3 - Deployment & Operations
Origin in ordering: 0.5.0 harness manager (Hangar M1; factory PR #372) — the package extraction of the doctor/channel gate checks
Workspace home: packages/preflight
Targets: All (checks run against a harness directory, not inside a bundle)
Test layers: T1, T3

## Purpose

Typed pre-spawn health checks: `will not boot: SLACK_SIGNING_SECRET unset` instead of a relayed stack trace. One report composed from seven areas, in composition order — **spec → credentials → channels → mcp → ports → bundle → durability** — with three levels (`info`, `warn`, `blocking`) and a stable `id` per item so a UI can acknowledge or deep-link an individual finding.

It backs `crewhaus harness preflight`, the Hangar console's expandable preflight panel, and the gate in front of **every** supervised spawn.

## Boundaries

Owns:

- **`credentials.ts`** — the provider env-var matrix over the **union** of every model a spec can route to: every `models:` profile, `agent.model`, `model_fallbacks`, `model_tiers`, `model_pool` candidates, the evaluation judge and its panel, and the budget degrade model. A spec that falls back to a provider you have no key for is a boot failure waiting for the first fallback, not a healthy harness.

  Since v0.6.0 the union **resolves `$profile` references** before it maps a slot through the model grammar — in the walk *and* in `extractSpecModel`, the tolerant single-slot reader whose result decides which provider's credentials `crewhaus doctor` and the `run` preflight demand. This is the one reference-resolution site outside the compiler: it reads spec text rather than IR, so without it an unresolved `$fast` would reach the grammar parser, throw, and hand every spec that adopts the registry a wrong credential verdict. The tolerant contract is unchanged — anything it cannot resolve (an unknown profile, a profile whose model is a compile-time sentinel) still yields `undefined`.
- **`channels.ts`** — the channel daemon's boot-gate secret env refs, checked offline as pure env presence, against **the exact set the compiled daemon exits 2 on**. That equality is what makes this the one unforceable area in the supervisor's gate: a manager cannot honestly wave through a check whose failure the child itself refuses to start on.
- **`mcp.ts`** — a dry-run of boot-time MCP secret-ref resolution, plus lint for `$…` literals the transports never expand and for credentials pasted inline.
- **`ports.ts`** — bindability of requested and declared ports.
- **`bundle.ts`** — spec-vs-`dist` freshness. The default is an approximate mtime heuristic, with a comparator seam so a caller holding the compiled bundle's provenance stamp can answer exactly instead. An unstamped bundle is reported as unstamped, never as stale.
- **`durability.ts`** — warn-level footguns: no dedup store on a channel daemon, live credentials with no budget cap.
- **`secret-grammar.ts`** — the shared `$VAR` / secret-ref grammar, the credential-shaped-key regex, and the malformed-ref messages, so "is this a secret reference" has one answer across areas.
- **`types.ts`** — `PreflightItem` / `PreflightReport` / `buildReport`, and `isEnvSet`.

Does not own: the compiler-dependent channel-IR extraction (that stays in the CLI and is injected), the gate policy (`harness-supervisor`'s `evaluateGate` decides what `--force` and `--ack` may wave through), the rendering (`apps/cli/src/harness-cmd.ts` and the console each render the same report), or any remediation — every item carries a `remediation` string, and nothing here applies it.

## Inputs and Outputs

Inputs: a harness directory, a parsed spec (or its issues), an explicit `env` record, and optional injected compiler warnings and comparator seams.

Outputs: a `PreflightReport` — `ok` (no blocking items), `blocking` (the subset), and `items` in composition order — plus the per-area `*Check` builders for callers that want one area.

## Dependency Notes

Depends on `@crewhaus/errors`, `@crewhaus/spec`, `@crewhaus/model-router` (to resolve what a spec can route to) and `@crewhaus/mcp-host`. **Every core function takes an explicit `env` and never reads `process.env`**; the one deliberate exception is the `preflightHarness` convenience wrapper, which defaults to the ambient environment.

That injection is not a testing nicety, it is the correctness property. The supervisor evaluates preflight against the **merged spawn env** — the harness `.env` chain *underneath* `process.env`, the same precedence `buildSpawnEnv` gives the child. An earlier version layered them the other way and could pass a check against a value the daemon would never see: the exact inversion of "it passed preflight and then died on a missing key". Both sides now delegate to the same function, with a test pinning them together.

The doctor credential checks and the `channel provision|verify` gate checks were lifted here **message-identical**, so `crewhaus doctor` and `crewhaus channel verify` say what they always said.

## First Implementation Slice

Shipped in factory PR #372 as an extraction with `harness-registry` and `harness-inventory`; wired as the supervisor's spawn gate in #373, and fixed in #375 to evaluate the merged spawn env rather than the manager's own.

## Study References

`doctor`'s environment probes (the messages this preserves), `target-channel-bot`'s boot gate (the exit-2 set the channels area mirrors), and `bundle-manifest`'s `crewhaus: { specHash, compiledWith }` stamp, which is the exact answer the freshness heuristic exists to be replaced by.

## Validation Plan

Catalog tests: T1, T3. Primary risks: **a false pass** — a check that consults an environment the child will not receive, or a channel set that has drifted from the daemon's own gate — and **a false blocker**, which teaches operators to reach for `--force` reflexively.

Definition of done: tests are green, no core function reads `process.env`, the channels area is asserted equal to the compiled daemon's boot-gate set, and item ids are stable across runs so acknowledgements survive.
