# hangar-ui

Status: implemented and tested
Dependency phase: 18 - Deployment & studio
Catalog layer: F3 - Deployment & Operations
Origin in ordering: 0.5.0 harness manager (Hangar M1–M4; factory PRs #372–#376)
Workspace home: packages/hangar-ui
Targets: All (the browser half of the manager console)
Test layers: T1

## Purpose

The Hangar console's static UI: one HTML shell, one CSS file, and a set of hand-written browser ES modules, embedded as text and exported as a serve-path map for `@crewhaus/hangar-server` to host. There is no build step, no framework, and no npm dependency — the `assets/` tree IS the app, loaded by the browser exactly as checked in.

M1 was read-only over harness state. M2 made the console a driver — start, stop, restart, drain, watch a run live, poke scheduler lanes, settle approvals and review items — with every action showing the exact CLI command it runs. M3 added eight harness tabs and three fleet screens over the 178-route contract.

## Boundaries

Owns:

- **`src/index.ts`** — the embed map. Every asset is imported `with { type: "text" }`, **statically**, because `bun build --compile` embeds only what the import graph references statically; a runtime `readFileSync` of a package-relative path resolves inside the compiled binary's virtual filesystem where the files were never placed, and bricks the CLI at boot. A unit test pins the embed map and the `assets/` tree in lockstep — every file present, every relative import resolving inside the map, every module transpiler-clean — so the zero-build path still has a compile-shaped safety net.
- **`assets/js/app.js`** — the auth bootstrap: match `#t=<token>` off the fragment, `decodeURIComponent` it into `sessionStorage`, strip it with `history.replaceState`. Any 401 swaps in a token-paste screen naming the token file.
- **`assets/js/routes.js`** — all 178 M3 routes as **pure data** tagged with a group. `api.js` builds one client wrapper per key from that map instead of hand-writing 178 of them; a hand-written name wins a collision, so the M1/M2 verbs keep their bespoke wrappers. The same table is what the server's contract test asserts set-equality against.
- **The hash router and its screens** — `#/` (Library), the fleet screens `#/{runs, approvals, review, activity, credentials, feedback, thredz}`, `#/h/<id>` and its tabs `{spec, runs[/<runId>], schedulers, sessions, evals, memory, data, feedback, costs, creds, channels, security, thredz, deploy, inspect, dev}`. The eight M3 tabs capture their trailing segments generically, so a sub-screen inside one never needs a new route case.
- **The live run feed**, hand-rolled over `fetch` rather than `EventSource`, for one reason: `EventSource` carries no bearer header.
- **`emptyState(message, verb)`** — the consumer of the server's `{present, note, verb}` triple, so an empty panel says why it is empty and which CLI verb fills it.

Does not own: any data (every screen reads the server), authentication policy, the route contract's authority (the server's `M3_ROUTES` is the peer table), or plugin execution — a plugin pane renders in an iframe with `sandbox="allow-scripts"` and the UI strips `allow-same-origin` again on the way in.

## Inputs and Outputs

Inputs: `hangarAssets` is consumed by the server as a serve-path map; the app itself consumes the JSON API and the SSE run feed.

Outputs: `hangarAssets` (`Readonly<Record<string, {body, contentType}>>`, keyed by serve path — `"/"` is the SPA shell, everything else under `/assets/…`), plus `CONTENT_TYPES` and `contentTypeFor()`.

## Dependency Notes

No runtime dependencies at all — that is the point. It is a leaf package whose only consumer is `apps/cli` (which hands the map to the server) and, transitively, the compiled `crewhaus` binary. `crewhaus hangar serve --smoke` check 2 (`GET / serves the embedded UI shell`, asserting the body contains `Hangar — CrewHaus`) is what proves the embedding survived `--compile`.

Not covered by the `windows-supervision` CI job, which runs the supervisor and server suites only.

## First Implementation Slice

Shipped with the four Hangar milestones (#372–#375): M1 the read-only library/detail screens, M2 the driving surfaces and the run console, M3 the eleven view modules (`views/{spec-edit, memory-fabric, evals-lab, data, feedback, creds, channels, security, thredz, inspect, runtime}.js`), M4 onboarding, the omnibox, notifications and the fleet health board.

## Study References

`@crewhaus/ui` (the DOM-only, `textContent`-rendering discipline it inherits — a harness can never inject markup into the page), `trace-viewer` (the transcript vocabulary the run console reuses), and `default-skills`' compiled-binary embedding bug, which is why every asset import here is static.

## Validation Plan

Catalog tests: T1. Primary risks: **an asset that exists on disk but not in the embed map** (the compiled binary then 404s a module the browser needs, at boot, with no build step to catch it), and **route-table drift** against the server.

Definition of done: the embed-map lockstep test is green, the server's contract test agrees with `routes.js` key-for-key, and `hangar serve --smoke` finds the shell marker in the compiled binary's response.
