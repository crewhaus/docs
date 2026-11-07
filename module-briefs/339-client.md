# client

Status: planned
Dependency phase: 19 - Plugin SDK
Catalog layer: F5 - Plugin SDK & Extension
Origin in ordering: 0.9.0, the Home release ([#522](https://github.com/crewhaus/factory/issues/522))
Workspace home: packages/client
Targets: none; an SDK for applications outside a harness (Node 20 or later, Bun, browsers and Workers)
Test layers: T1, T2, T3

## Purpose

Let an application start a harness run, follow it and read its result, through a typed client of `crewhaus.hangar.v1`, the API of Home, the 0.9.0 name for Hangar's web console. An app's server creates a run and streams its progress to a page that holds only a scoped token, and it can receive signed webhooks when the run's state changes.

It is a package of its own because an embedding SDK must load on plain Node, browsers and Workers with zero runtime dependencies. It is the first package here meant for plain Node, so it joins a class the release checks: relative imports carry `.js`, no Bun API is used, it type-checks under NodeNext, and CI installs its packed tarball into an empty project and imports it with `node`.

## Boundaries

Owns:

- **`createCrewHausClient`.** Discovery and negotiation, runs (create, get, events, result, cancel, resume), decisions, artifacts and browser tokens. Every POST carries an `Idempotency-Key` header, generated when the caller gives none, so the client's own retries never run anything twice, and it retries only errors marked retryable. Each error code maps to a typed error such as `AlreadyDecidedError` or `UnsupportedError`.
- **Resumable events.** A fetch-based SSE reader resumes with `Last-Event-ID` and reports missed events, resets and pauses. `result()` resolves only on a finished run. It refuses, with a remedy, a Hangar older than 0.9.0 and a managed harness's own `crewhaus.v1` gateway, which stays that daemon's protocol.
- **Subjects.** Only a service token holding `h.subject.assert` can set a run's subject, for one-shot jobs. It travels in the run's 0600 input file, never argv, the environment or a row. A managed harness answers `unsupported` and runs nothing, because its subject comes only from the verified token.
- **Scoped browser tokens.** The server mints an origin-bound token: read-only, for the runs and acts it lists, valid at most 15 minutes, unable to create runs, decide or list. Issuance is sealed, and the page holds only that token. A `progress` view of a run's events carries step ids and tool categories, never tool names, inputs or outputs, so an end user can watch it.
- **`./webhooks`.** `verifyWebhook` checks Standard Webhooks signatures (`webhook-id`, `webhook-timestamp`, `webhook-signature`) with WebCrypto, a separate implementation held to `tool-notify`'s signer by a shared test vector. Payloads carry ids and states only, so a receiver fetches details with its own token and every read is authorized again.
- **`./elements`.** `<crewhaus-run>`, `<crewhaus-result>`, `<crewhaus-decisions>` and `<crewhaus-inspector-link>`: light-DOM custom elements that take the page's styles, hold only a scoped token, never decide, build the DOM without `innerHTML` and announce progress politely.

Does not own: the operations, schemas and error codes it speaks (`control-contract`, brief 322, whose dependency-free `.` entry it bundles at build); the server side of every route, the scoped token's checks and CORS (`hangar-server`, brief 301; `access-control`, brief 323); delivering webhooks and subscriptions (`attention`'s `./notifications`, brief 331, and `tool-notify`'s signer); classifying run input (the run takes it in at TrustOrigin `api`, so the client classifies nothing); and a Python client, which is not part of this release.

## Inputs and Outputs

Inputs: an endpoint and a token, which the caller supplies from its own server's environment; run requests whose subject comes from the server session, never the browser; and webhook bodies and headers.

Outputs: typed runs, events, results and decisions; artifacts, covering a run's output and proof bundles, its pull requests once they are recorded, and spilled tool output through the inspector's export; a minted scoped token; a verified webhook, or a typed refusal. `runs.result` gives the status, output, evidence, cost and `inspectionUrl`. An unpriced cost stays unknown, never `$0`.

## Dependency Notes

No runtime dependency, and a test asserts the package declares no `dependencies`. It bundles `control-contract`'s dependency-free `.` entry at build, so its JavaScript and types are self-contained. That entry is the part it bundles: the authorizer and consequences templates of brief 322's planned `./delegation` stay out.

Dev dependencies: `@crewhaus/control-contract`, `@crewhaus/access-control`, `@crewhaus/hangar-server`'s public API and `service-fakes` (brief 327), which is never published.

## First Implementation Slice

`feat(client): a typed SDK, scoped browser streaming and signed webhooks for embedding a harness` adds the package, after the Node-loadable publish class, event ids and the inspector's export. Later PRs extend it as the API grows, and a contract suite holds every first-party client to one contract. This brief lands in crewhaus/docs before the package does.

## Study References

`packages/gateway-protocol/src/index.ts` (the `crewhaus.v1` header this replaces for applications); `packages/tool-notify` (the webhook signer); `packages/hangar-ui/src/embed-suite.ts` (the markup ban its elements share); the Standard Webhooks specification; and [#522](https://github.com/crewhaus/factory/issues/522).

## Validation Plan

Catalog tests: T1, T2, T3. Primary risks: **a retry that runs twice**: every POST carries its header, and a fixture Home proves one run per key; **a page holding more than it should**: a scoped token is refused from any other origin, and for creating runs, deciding and listing; **a dependency sneaking in**: a test on the package manifest, and the packed tarball imported with `node` in an empty project; **a forged webhook**: a bad signature, a stale timestamp and a redelivery each behave as the shared vector says.

Definition of done: `bun test packages/client/src` green; the three examples (server, browser streaming and no UI) run in [crewhaus/demos](https://github.com/crewhaus/demos) CI against a fixture Home; the packed tarball loads on Node 20; and Home, the CLI, the Slack app and the SDK show the same run and decision ids in the contract suite.
