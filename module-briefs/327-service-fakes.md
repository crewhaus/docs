# service-fakes

Status: planned (test-only; never published)
Dependency phase: 1 - Foundations
Catalog layer: F2 - Compiler & Codegen
Origin in ordering: 0.9.0 ([#507](https://github.com/crewhaus/factory/issues/507), [#513](https://github.com/crewhaus/factory/issues/513))
Workspace home: packages/service-fakes
Targets: none (test-only)
Test layers: T1, T2

## Purpose

Loopback fakes of the services a harness talks to: a model provider, Slack, GitHub, Thredz and SMTP. It is a dependency-free leaf, so any package's own tests can start a fake with no cycle through hangar-server. `hangar-e2e` (brief 326) depends on hangar-server, which is why the fakes cannot live there.

## Boundaries

Owns:

- **`fakeModel`.** The Anthropic Messages API, plain or as SSE, with replies scripted by prompt digest, declared prices and unpriced models, and 401, 429 and 529 scripts. It counts requests and records masked request bodies.
- **`fakeSlack`.** A sink for the Web API calls harnesses make, and a client that posts signed Events API and interactivity deliveries, with duplicate `event_id` replays.
- **`fakeGitHub`.** A REST subset, smart HTTP through `git http-backend` over a temporary bare repository, and signed webhook deliveries carrying `X-GitHub-Delivery` and `X-Hub-Signature-256`.
- **`fakeThredz`.** The REST subset hangar-server calls.
- **`fakeSmtp`.** It records messages.

Every fake binds `127.0.0.1` on port 0 and is attached only through a seam that already exists, so the code under test is the code that ships. Keys are generated at runtime, and no secret-shaped literal is committed. Later PRs extend it: the integrations PRs add the GitHub manifest, App and OAuth endpoints and the Slack OAuth, manifest and install endpoints.

Does not own: browser journeys and the seeded fleet (`hangar-e2e`, brief 326); the real clients it stands opposite (`adapter-anthropic`, `channel-adapter-slack` and the rest), or any new seam in them; and hangar-server's own test scaffolding.

## Inputs and Outputs

Inputs: a test's script for a fake (replies, prices, faults, deliveries to send) and whatever a real client sends to the fake's address.

Outputs: the fake's address, responses in the service's wire shape, recorded requests with their bodies masked, request counts, and signed deliveries posted to the receiver a test names.

## Dependency Notes

None, on purpose: it is a leaf with no workspace dependency, so any package's tests can import it. Importers use a devDependency with `workspace:*`, never a semver range.

The name rule and the guards are the ones in brief 326: an unscoped name with no `crewhaus` prefix, plus `"private": true` and `"version": "0.0.0"`, held by one guard file under `scripts/`. No publishable package may name it in `dependencies`, `peerDependencies` or `optionalDependencies`. Its modules avoid the test-only file names `scripts/test-only-modules.test.ts` watches.

## First Implementation Slice

Lands with `hangar-e2e` in one change: the five fakes, and fidelity tests in `adapter-anthropic` and `channel-adapter-slack` that drive each fake through the real client. The root `test` script gains the package's filter, so its own tests run in `ci`.

## Study References

The real clients in `adapter-anthropic` and `channel-adapter-slack`, whose requests the fakes must accept unchanged; `packages/hangar-server/src/testkit.ts` (test scaffolding that stays unexported from its package, the reason this is a package of its own); `scripts/test-only-modules.test.ts`; the Anthropic Messages API, Slack's request-signing and GitHub's webhook documentation; [#507](https://github.com/crewhaus/factory/issues/507) and [#513](https://github.com/crewhaus/factory/issues/513).

## Validation Plan

Catalog tests: T1, T2. Primary risk: **a fake that diverges from its service**, so a test passes against something no real service does. Fidelity tests drive each fake through the real client, and recorded bodies are masked.

Second risk: **a fake that listens beyond the machine.** A bind test checks the listening address itself, because a test that only fetches the address it bound cannot tell loopback from a wildcard.

Definition of done: `bun test packages/service-fakes/src` green, in the root suite; the fidelity tests green in the two client packages; and no secret-shaped literal in the package.
