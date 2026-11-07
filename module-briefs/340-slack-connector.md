# slack-connector

Status: planned
Dependency phase: 15 - Channels
Catalog layer: R13 - Channels & Messaging
Origin in ordering: 0.9.0, the Home release ([#522](https://github.com/crewhaus/factory/issues/522))
Workspace home: packages/slack-connector
Targets: All (it operates compiled harnesses; it is not embedded in one)
Test layers: T1, T2, T3, T8

## Purpose

One Slack app per Home, the 0.9.0 name for Hangar's web console, that reaches every harness. From Slack a team can see what needs them, settle a decision, pause a harness or ask one a question, under the same members, roles and decision claims as every other surface.

It is a package of its own because Home's Slack app sits above `control-service` and `attention`, while `channel-adapter-slack` stays the light per-daemon adapter that bundles import. The two have different hosts and hold different credentials. Per-harness bots keep working for conversation.

## Boundaries

Owns:

- **The install state machine.** Resumable steps over `service-setup`'s Slack client, each showing its state in words: a public address, creating the app from a manifest with a configuration token used once and never stored, installing through OAuth with a single-use state, then linking an account and sending a test that reads "Verified end to end" only when a click comes back through the public address. A workspace whose admins must approve leaves the step "Waiting for approval", and resuming reuses the app already created.
- **The receivers.** `/hooks/slack/events`, `/actions` and `/commands` check Slack's signature, accept only installed workspaces and this app, claim each delivery once and write the receipt before they answer, with no network call first. The one unsigned request answered is Slack's URL-verification challenge while an app is being created.
- **Authority.** The actor is the member linked to `slack:<team>:<user>`. An unlinked person is asked to link, and nothing runs. Each command is an operation with its permission, every decision goes through the decision service at the item's required class, and the app never writes a settings rule. Rules are never turned on from Slack, because a card cannot show the full consequences.
- **The command vocabulary.** `/crewhaus` is closed and parsed without a model, in a direct message or a channel connected for commands: `status`, `inbox`, `pause`, `resume`, `ask <harness> <question>`, `digest`, `link` and `help`. `@CrewHaus <harness> <question>` is `ask`, the one command that starts work. Replies come from the receipt, and an `ask` to a harness whose host is down gets "Not done", with no run created.
- **Cards.** Inbox items sent through `attention`'s notifications sender, as direct messages to linked members who can decide; channels get summaries, never buttons. A card offers the choices only when the whole input and every place it acts fit the card, and otherwise links to Home. A click on a card whose evidence changed decides nothing, and a card decided elsewhere says who decided where.
- **Answers.** An `ask` answer is posted where the question was asked, masked and shortened, only when the run's `sensitivity` is `internal`; otherwise the reply says the answer is only in Home, with a link.
- **Discuss, App Home and linking.** Discuss adds a note to an item's thread. Linking uses a single-use request bound to the Slack workspace and user, valid 15 minutes, and the confirm page warns that anyone using that Slack account will decide as you. Unlink is one click.
- **Classification, once in each place.** `ask` and `@CrewHaus` text is classified at TrustOrigin `api` where the run takes it in, and nowhere on the way. A Discuss note kept for people is classified at `channel` here, in Home, before it is stored; a malicious one is stored as the redaction notice and never sent to a harness. Text only parsed as a command reaches no model and is not stored. Authentication is not classification: the signature says who sent it, and the member's role decides.

Does not own: the per-daemon adapter and the planned `./permissions` table of scopes (`channel-adapter-slack`, which every Slack path reads); the `ChannelAdapter` contract (`channel-adapter-base`); the inbox model and notification planning (`attention`, brief 331); the receiver listener and the ingress registry (`control-service`, brief 325); secrets at rest (`secrets-manager`'s Home secret store); run input classification (the `api` sites of the run); and the GitHub App (`github-app`, brief 338).

## Inputs and Outputs

Inputs: signed Slack events, actions and commands; a configuration token, typed once; a linked member's identity; inbox items from `attention`; and receipts from the operations it starts.

Outputs: inbound receipts and claims; `runs.create` for `ask`; decisions through the decision service; cards, App Home views and fixed-template posts with a text fallback; and links to Home that carry no bearer and need sign-in. States are words, never colour or emoji alone. Its secrets go into the Home secret store and never come back out: responses say only whether each is configured. With Home stopped the receivers are off and Slack's request simply fails; the installed resident service keeps them on.

## Dependency Notes

Depends on `@crewhaus/control-contract`, `@crewhaus/control-records`, `@crewhaus/control-service`, `@crewhaus/access-control`, `@crewhaus/attention`'s `./notifications`, `@crewhaus/channel-adapter-slack`, `@crewhaus/service-setup`, `@crewhaus/secrets-manager` and `@crewhaus/boundary-classifier`. It runs in Home and the CLI and never enters a bundle (a guard test). No runtime dependency is added beyond these, and its receiver joins `crewhaus doctor --philosophy-alignment`'s boundary list with a drift row in the PR that adds it.

Harnesses never depend on it: execution, decisions and controls keep working with Home stopped.

## First Implementation Slice

`feat(slack-connector): one Slack app per Home reaches every harness, with linked authority and honest replies` adds the package, with the `ChannelAdapter` move and the permission table, after the inbox and notifications. Later PRs extend it as areas add commands. This brief lands in crewhaus/docs before the package does.

## Study References

`packages/channel-adapter-slack/src/index.ts` (`verifySlackSignature` and the adapter it stays above); `packages/service-setup/src/plan.ts`, `apps/cli/src/channel-provision.ts` and `packages/hangar-server/src/channels-ops.ts` (the three places scopes are listed, which one table replaces); `packages/target-channel-bot/src/index.ts` (the per-harness approval path); and [#522](https://github.com/crewhaus/factory/issues/522).

## Validation Plan

Catalog tests: T1, T2, T3, T8. Primary risks: **a click deciding what the clicker never saw**: choices appear only when the whole input fits, and a changed card decides nothing; **an unlinked or demoted person acting**: the deny matrix covers every operation with really minted credentials; **a replayed delivery**: a duplicate `event_id` runs one effect; **a slow answer**: the receiver answers from the receipt with no network call first; **a scope the adapter needs and the manifest lacks**: a test fails when the adapter calls a method whose scope is missing.

Definition of done: `bun test packages/slack-connector/src` green; a journey against a fake Slack installs, links, decides from a card and asks a question; `crewhaus doctor --philosophy-alignment` exits 0 with the receiver listed; and a real Slack workspace run stays on the pre-release checklist.
