# service-setup

Status: implemented and tested
Dependency phase: 18 - Deployment & studio
Catalog layer: F3 - Deployment & Operations
Origin in ordering: post-0.6.0 (factory PR #452) — the operator-tier counterpart to `channel provision`, which is emit-and-instruct for Slack by design
Workspace home: packages/service-setup
Targets: channel (Slack + tunnel), plus any memory shape carrying a `thredz:` block
Test layers: T1, T3

## Purpose

Create the external services a spec asks for but cannot create itself: a Slack app, a Cloudflare named tunnel with a public hostname, and a Thredz wiki space. It backs `crewhaus services setup`.

A spec declares *that* it wants a Slack channel, a public URL and a hosted wiki. Everything on the other side of those declarations — an app with the right scopes, a hostname that resolves, a space the key can write to — has been a click-path through three consoles, repeated per harness. This is that click-path as code, driven entirely off the spec.

## Boundaries

Owns:

- **`target.ts`** — the spec → `SetupTarget` derivation. This is what keeps the package convention-free: the credential variable **names** come from the spec's own `$VAR` refs, because `channel-adapter-slack` never reads a credential from the environment (the daemon passes them in as constructor config), so the names are the author's choice and nothing may assume otherwise. Reading is tolerant plain YAML, not `parseSpec` — a spec failing a strict rule unrelated to services must still be provisionable.
- **`cloudflare.ts`** — find-or-create a tunnel (always `config_src: "cloudflare"`; a `local` tunnel is rejected loudly, since there is no API to migrate one), fetch the connector token, read-merge-write the ingress, and read-then-write the DNS CNAME.
- **`slack.ts`** — build the manifest, validate it before spending the Tier-1 create budget, create or update the app, and complete the OAuth v2 exchange.
- **`thredz.ts`** — list-first, 409-tolerant space find-or-create, distinguishing a recoverable slug conflict from the one-individual-space-per-key limit.
- **`responder.ts`** — the stand-in listener that answers Slack's `url_verification` challenge and catches the OAuth callback.
- **`env-file.ts`** — the credential write-back, semantics-identical to `hangar-server`'s `upsertEnvVar` so the console and this command agree on what "already set" means.
- **`spec-edit.ts`** — the one spec field setup writes, `thredz.space`, edited through the `yaml` CST so comments and key order survive.
- **`plan.ts`** — step order, and what is persisted when.

Does not own: rendering (the CLI renders the returned changes), prompting (an injected `io` seam), the cloudflared connector's lifecycle (setup prints the install command and never shells out to `sudo`), and the daemon's own boot gate (`@crewhaus/preflight` owns that, and `channel verify` reports it).

## Inputs and Outputs

Inputs: a harness's `crewhaus.yaml`, the events port (the `PORT` env var — it has **no** spec field and so cannot be derived), a zone, and three short-lived provisioning credentials.

Outputs: `AppliedChange[]` in the shared `✓ / ~ / ✗` shape, plus follow-up lines for the two things a person must still do — run the connector, and approve the Slack install.

## Dependency Notes

Depends on `@crewhaus/errors`, `@crewhaus/preflight` (the shared `$VAR` secret grammar) and `yaml`. **No module reads `process.env` or calls global `fetch`** — both are injected, so the surface is testable without credentials.

The important boundary is a *credential tier*, not a package boundary. `channel provision` is deliberately emit-and-instruct for Slack: the spec carries only what the daemon uses, and none of it can call the manifest API. This package does not reverse that decision, it sits above it. An operator running setup holds provisioning credentials that are read once, used, and dropped; what lands in the harness is still only what the daemon needs.

## First Implementation Slice

The three clients, the spec derivation, the write-back, the responder, and `crewhaus services setup`, with `--dry-run` and `--services` narrowing.

## Study References

`channel-provision.ts`'s scope derivation (kept in lockstep — an app this command creates and an app that command describes must be the same app), `hangar-server`'s `upsertEnvVar` (the write-back semantics), and `doctor-fix.ts`'s plan/apply split.

## Validation Plan

Catalog tests: T1, T3. Primary risks, in order:

1. **Losing an unrecoverable secret.** Slack returns the signing secret exactly once, in the create response; no API reads it back. The credentials are therefore written to `.env` immediately on receipt, before anything that can fail, and a test asserts they survive a failed OAuth step.
2. **Clobbering a shared tunnel.** `PUT …/configurations` replaces the entire configuration, so the ingress write must be read-merge-write, and the catch-all must stay last — the API accepts a malformed list with `success: true` while the connector silently keeps serving the old one.
3. **Creating a duplicate app.** There is no API to list an account's apps, so re-run safety comes from the recorded app id in `.env`, which switches a second run onto the update path.

Definition of done: tests green, no `process.env` read, the Slack scope derivation asserted equal to `channel provision`'s, and every step idempotent on a second run.
