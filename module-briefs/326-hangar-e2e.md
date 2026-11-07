# hangar-e2e

Status: planned (test-only; never published)
Dependency phase: 18 - Deployment & studio
Catalog layer: F2 - Compiler & Codegen
Origin in ordering: 0.9.0 ([#507](https://github.com/crewhaus/factory/issues/507), [#513](https://github.com/crewhaus/factory/issues/513))
Workspace home: packages/hangar-e2e
Targets: none (it tests Home; never in a compiled harness)
Test layers: T1, T3

## Purpose

Browser journeys over the real console. This package starts Home (the Hangar console) against a seeded fleet, drives it in a real browser the way a person would, and fails when a journey, an accessibility check or a release check does not hold.

It is a package of its own because journeys need a browser and a running Home, which hangar-ui's DOM-free unit suite must never load. It sits beside `smoke-harness`: that one proves every shape compiles, and this one proves the console works once it is running.

## Boundaries

Owns:

- **Boot, through the public surface only.** It starts Home in-process with hangar-server's public `startHangarServer`, `createProcessLayer` and `makeFixtureHarness`, with recording notification sinks so no OS toast fires. A system mode runs the real CLI from source, or the binary `CREWHAUS_E2E_BIN` names.
- **Browser sessions.** A session signs in through the real boot path, and fails on a console error, a page error, a CSP violation, a failed asset or a script-execution sentinel, so a journey never passes over a page that broke along the way.
- **The journey suite and its registry.** Every journey is listed, and each area adds its own as its console screens land.
- **The accessibility toolkit.** An axe runner injected with `page.evaluate`, never `bypassCSP`; a keyboard walk; 320 px reflow; text spacing; reduced motion; target size; focus not obscured; and ARIA snapshots through `locator.ariaSnapshot()`.
- **The seeded fleet.** A generator that writes harnesses and their records through the real store libraries, never hand-rolled JSON.
- **Measures.** A navigation-depth crawler, request and module counts, and a benchmark for a reference machine.
- **Release checks.** Cross-surface contract tests, the release acceptance registry, the fault catalog, the pre-release checklist generator and the upgrade-and-revert script.
- **Three scripts.** `test` runs the DOM-free units and is part of the root suite; `test:browser` runs the journeys in the `ui-e2e` CI job; `bench` runs the benchmark.

Does not own: the loopback fakes of outside services (`service-fakes`, brief 327); hangar-server's `testkit.ts`, which stays unexported from that package (`packages/hangar-server/src/testkit.ts`, and the test-only excludes in `tsconfig.base.json`); the console itself (`hangar-server`, brief 301; `hangar-ui`, brief 302); hangar-ui's DOM-free unit suite; and the compile-time matrix over every shape (`smoke-harness`).

## Inputs and Outputs

Inputs: hangar-server's public start surface; a browser from Playwright; a fleet written through the store libraries; `service-fakes` for anything outside the machine; and, in system mode, the CLI or `CREWHAUS_E2E_BIN`.

Outputs: a pass or a failure per journey, with the reason; an accessibility report; request, module and navigation-depth counts; benchmark numbers from the reference machine; and the pre-release checklist.

## Dependency Notes

Dev only: `playwright` at the root pin, `axe-core` at an exact pin, and `workspace:*` on `hangar-server`, `hangar-ui`, `control-contract` (once that package exists), `service-fakes` and the store libraries it seeds through. Nothing depends on it.

The name rule has two locks. The package name is unscoped and has no `crewhaus` prefix, which `scripts/release-prep.ts` never stamps, and the manifest says `"private": true` and `"version": "0.0.0"`, which `scripts/publish-workspace.ts` skips. Three guards hold the rule, one `scripts/*.test.ts` file:

1. Every `"private": true` workspace package has a name release-prep does not stamp, and appears in the root `test` filters.
2. No publishable package names a private package in `dependencies`, `peerDependencies` or `optionalDependencies`.
3. Any workspace manifest, devDependencies included, names a private package only as `workspace:*`, so the name resolves only inside the workspace.

Its modules are named plainly, never `fixtures.ts`, `*-fixtures.ts`, `fixtures/`, `testkit.ts` or `test-helpers.ts`. Those names are kept out of every build, and `scripts/test-only-modules.test.ts` fails when non-test source imports one, private packages included. The journey suite and the seeders are ordinary modules.

## First Implementation Slice

Lands with `service-fakes` (brief 327) in one change that needs no earlier control-core PR: this package and its boot, session and seed modules; the `ui-e2e` CI job, sharded from the start into console smoke, the axe sweep and system journeys, each shard inside the time the `ci` job takes; a job-level condition that runs the axe sweep only when hangar-ui or hangar-server changes, and reports success when it skips; smoke journeys over the console as it is; a report-only axe baseline; `--filter 'hangar-e2e' --filter 'service-fakes'` in the root `test` script, so the DOM-free units stay in `ci`; and the three guards.

Later changes add the accessibility gate over the routes they retrofit, each area's journeys, the cross-surface contract tests, the release registries and the benchmark.

## Study References

`smoke-harness` (the compile-time matrix this complements); `startHangarServer`, `createProcessLayer` and `makeFixtureHarness` in `packages/hangar-server/src/index.ts`; `scripts/release-prep.ts` and `scripts/publish-workspace.ts` (which names are stamped, and which are published); `scripts/test-only-modules.test.ts`; [HANGAR.md](../HANGAR.md#one-state-tree-two-heads) (the console and the CLI as two heads over one state tree); [Playwright](https://playwright.dev/) and [axe-core](https://github.com/dequelabs/axe-core); WCAG 2.2; [#507](https://github.com/crewhaus/factory/issues/507) and [#513](https://github.com/crewhaus/factory/issues/513).

## Validation Plan

Catalog tests: T1, T3. Primary risk: **a journey that passes without the behaviour.** Sessions fail on console errors, page errors, CSP violations, failed assets and a script-execution sentinel, seeds go through the real stores, and Home boots through its public start surface, never a private test harness.

Second risk: **a test package reaching npm.** The two locks and the three guards answer it, and each guard is a test that fails on a manifest that breaks the rule.

Third risk: **a check that gets skipped because it is slow.** Each shard stays inside the `ci` job's time, and the axe sweep's condition reports success when it skips, so a required check never hangs on a job that did not run.

Definition of done: `bun test packages/hangar-e2e/src/unit` green, in the root suite; `test:browser` green in the `ui-e2e` job; the three guards green; and the package absent from every list the publish scripts build.
