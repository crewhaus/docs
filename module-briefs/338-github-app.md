# github-app

Status: planned
Dependency phase: 18 - Deployment & studio
Catalog layer: F3 - Deployment & Operations
Origin in ordering: 0.9.0, the Home release ([#522](https://github.com/crewhaus/factory/issues/522))
Workspace home: packages/github-app
Targets: All (it operates compiled harnesses; it is not embedded in one)
Test layers: T1, T2, T3, T8

## Purpose

Give a harness the narrowest GitHub credential that works. Each Home, the 0.9.0 name for Hangar's web console, registers its own private GitHub App for each account or organization it connects, keeps the App's key, and mints short-lived installation tokens for one repository at a time. Nothing goes to CrewHaus.

It is a package of its own because the App key's custody and lease minting must never enter a harness: `tool-codehost`, on the bundle side, only reads leases. Merging it with `slack-connector` would save a name, but nothing links them and they hold different credentials.

## Boundaries

Owns:

- **The manifest flow.** Registers the App for the owner's account, an organization or a GitHub Enterprise Server. Its key and secrets come back once, and Home stores them before anything else. Permission presets: read only; read and open pull requests; comment on issues. Never administration, workflows or secrets. An organization that must approve leaves the step "Waiting for approval" until GitHub's installation event or the scheduler's check moves it.
- **App JWTs.** RS256 signed with WebCrypto, GitHub's PKCS#1 key wrapped as PKCS#8, so no dependency is added.
- **Installation tokens and leases.** A token narrowed to one repository and the binding's permissions, valid at most an hour, written per binding under `.crewhaus/run/control/keys/codehost/` (mode 0600, atomically). The scheduler refreshes it, and no read ever does. `crewhaus daemon start` mints a first one. Unbinding, uninstalling or disconnecting deletes it and revokes the token at GitHub.
- **Webhooks.** `/hooks/github/<app>` checks `X-Hub-Signature-256` in constant time, claims `X-GitHub-Delivery`, writes the receipt and answers 202. Consumers register by event kind in a consumer registry: project sources, pull-request outcomes, connection health. Without a public address Home polls instead, and says so.
- **Classification at `source`.** Issue, pull-request, comment and review text from webhooks is classified at TrustOrigin `source`, field by field, in Home, before any consumer stores it. A malicious field is stored as the redaction notice, and its source marked; a suspicious one is kept and flagged. Authentication is not classification: a valid signature says who sent a request, not what its text is.
- **Callbacks.** The manifest, setup and user sign-in callbacks are public `GET /oauth/github/…` routes on the main listener. Each takes a single-use state bound to the member who started the flow and to the Host, and exchanges its code on the server. They carry a code and a state, never text for a model.
- **User linking.** The App's user sign-in reads the id, revokes the token at once and records a `github:<id>` link, so reviews and merges are attributed to a member.

Merging is not a preset's to grant. Nothing here merges a pull request. GitHub lets contents and pull-request write merge, so the step that grants them asks the owner to turn on branch protection that requires a review. Each merge stays a person's decision, or an owner's separate rule (brief 333), and a delegated merge still waits for that review unless the owner lets the App bypass it on GitHub.

Does not own: the credential seam a tool reads, the planned `./credentials` entry on `@crewhaus/tool-codehost`, with its environment and lease providers; the publish engine and git custody (`tool-work`, brief 333); repository connections and the binding store (`control-records`, brief 324); where secrets rest (`secrets-manager`'s Home secret store); the receiver listener and ingress registry (`control-service`, brief 325); the `tool_config.codehost.installation` form ([#514](https://github.com/crewhaus/factory/issues/514)); and the Slack app (`slack-connector`, brief 340).

## Inputs and Outputs

Inputs: an owner's choice of account and preset; the manifest conversion's response, once; webhook deliveries; a repository binding from the connection store; and the App's key from the Home secret store.

Outputs: an App record; content-free installation and sync state; a lease per binding; revocations; `github:<id>` links; and classified webhook fields handed to registered consumers. Values go in and never come back out: responses say only whether each credential is configured. Each operation has a real CLI twin or none with its reason; a GitHub sign-in needs a browser, so it has none.

## Dependency Notes

Depends on `@crewhaus/control-contract`, `@crewhaus/control-records`, `@crewhaus/access-control`, `@crewhaus/secrets-manager`, `@crewhaus/boundary-classifier` and the types of `@crewhaus/tool-codehost`'s planned `./credentials`. No runtime dependency beyond these: WebCrypto and `fetch` cover the protocol. It runs in Home and the CLI and never enters a bundle (a guard test), and its receiver joins `crewhaus doctor --philosophy-alignment`'s boundary list.

Installation mode is opt-in. In it, codehost calls need a running Home or resident service to refresh the lease once it expires; `token_env` keeps working with no such dependency, for harnesses without Home and for other hosts.

## First Implementation Slice

`feat(github-app): each Home registers its own GitHub App, and harnesses use short-lived installation tokens` adds the package, after repository connections and the `source` origin. Later PRs extend it, as consumers register for webhook events. This brief lands in crewhaus/docs before the package does.

## Study References

`packages/tool-codehost/src/net.ts` (the token path that stays); `packages/secrets-manager` (the file backend beside a new keychain backend); `apps/cli/src/approval-gate.ts` (reading GitHub's checks fail-closed); GitHub's App manifest flow; and [#522](https://github.com/crewhaus/factory/issues/522).

## Validation Plan

Catalog tests: T1, T2, T3, T8. Primary risks: **the App key reaching a harness**: a guard keeps the package out of every bundle, and a test plants each credential shape and asserts the masker recognizes it; **a token wider than its binding**: a fake GitHub records the narrowed request and refuses anything else; **a forged or replayed delivery**: a bad HMAC, a repeated delivery id and a stale callback state are each refused; **a planted instruction in an issue**: a malicious field reaches no consumer as text.

Definition of done: `bun test packages/github-app/src` green; the JWT signer verifies against keys generated at run time, never a fixture key; a journey against a fake GitHub registers an App, mints a lease, receives a signed webhook and revokes on disconnect; and a live GitHub App run stays on the pre-release checklist.
