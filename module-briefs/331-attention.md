# attention

Status: planned
Dependency phase: 18 - Deployment & studio
Catalog layer: F3 - Deployment & Operations
Origin in ordering: 0.9.0, the Home release ([#517](https://github.com/crewhaus/factory/issues/517))
Workspace home: packages/attention
Targets: All (it operates compiled harnesses; it is not embedded in one)
Test layers: T1, T3, T8

## Purpose

One inbox for every decision and problem across harnesses, and the notifications that bring people to it. Tool approvals, the review queue, the Advisor, graph pauses, protected-environment decisions and failed runs each keep their own records; this package folds them into one item model, so a person sees what needs them in Home, the 0.9.0 name for Hangar's web console, in the CLI or in Slack, and answers once from any of them.

It is a package of its own so that the CLI, Home, the resident service and the Slack connector share one inbox model. It sits above `control-service`, which never imports an area, and below `hangar-server`. It stores only what no source holds: assignment, snoozes, actions on problems, threads and deliveries. Everything else is a fold over the sources.

## Boundaries

Owns:

- **The read-only adapters (`.`).** One per decision and signal source, each reading without a compacting read and without writing, plus `registerProducer` for kinds other areas add. Kinds are a closed, versioned list in `control-contract`.
- **The item.** It names the affected work, why it matters now, the evidence, each choice with its effect and the class it needs, a recommendation, a deadline only if one is real, what happens if no one answers, and its state. Titles come from closed templates. Tool input, question text, graph state and threads are body fields, returned only with `h.read.content` and masked. Recommendations are deterministic, and owner-class items carry none.
- **Ids that follow the evidence.** Derived `att_` ids from the source record or a cause key, so every head computes the same id without writing. An evidence revision moves only with new evidence, and dismissals, dedupe and notifications bind to both.
- **Cause keys and re-ask folding.** Items merge across harnesses only on a structured cause: a credential source, a connection, a provider limit or a deployment. Message similarity groups only within one harness, because a wrong merge would hide a distinct failure. Pending approvals with one ask digest in different scopes fold into one item; deciding it decides each record separately, and each grant stays in its own scope.
- **Lifecycle derivation.** Open, then assigned or deferred, then resolved, expired or superseded, computed by a pure `effectiveAttentionState` over the decision records.
- **Resolve routing.** `attention.resolve` sends each choice to its source's sanctioned writer: the decision service, a claim plus the resolver's identity for review entries, a resume claim for graph pauses, the Advisor's ledger for problems. A choice that acts also needs that act's permission. Assign names a member who holds the item's class and grants nothing.
- **Notifications (`./notifications`).** The delivery planner, immediate only for a decision this member can make and for an operational failure, and a daily digest otherwise. Per-member preferences, IANA quiet hours correct across daylight saving, digests and result cards, escalation, and durable dedupe and claims keyed by item, evidence revision, member and sink, so deliveries are deduplicated. A retry can redeliver, and a webhook receiver drops a redelivery by its `webhook-id`. The scheduler task runs in the resident service or a foreground Home. This export carries what brief 234 once named.
- **The sinks.** In-app; the Home host's OS notifier; email; Slack through a sender the connector injects (direct messages to linked members, summaries only to a channel); and a signed, owner-only webhook. Decision and question cards in Home and Slack show the masked tool input or question to a member whose role reads content. Email, OS and webhook payloads stay content-free: closed templates, ids, counts and a link that needs sign-in, never a bearer or profile data. Recipients are filtered by `h.read` first.
- **Other kinds.** Delegation's item kinds (`merge-ready`, `rollout-ready`, `delegation-alert`) register through `registerProducer` from their own PRs, as do quality repairs, `source-conflict`, profile corrections and budget items.

Does not own: `RequestGuidance` and the answers it carries (`tool-consult`, brief 309, with `runtime-core`'s seam); the Home-level Slack app, its receivers and its command vocabulary (`slack-connector`, brief 340); delegation rules and the delegated path (`control-service`, brief 325); the action classes and authority lowering (`permission-engine`'s planned `./authority`, with the spec delta [#514](https://github.com/crewhaus/factory/issues/514)); the source records themselves (`session-store`, `feedback-distill`, `harness-advice`, `checkpoint-store`, `deployment-controller`); the projector that builds its disposable read model (brief 325); and the screens (`hangar-ui`).

## Inputs and Outputs

Inputs: a harness's source records, read without writing; the asking member and the access snapshot; a choice with its evidence revision; and, for delivery, each member's preferences and the owner's shared sink settings, whose secrets sit in the Home secret store of [#522](https://github.com/crewhaus/factory/issues/522).

Outputs: items, in three bands (Decide, Fix, Consider); a disposable projection document and decision-search documents; receipts from the writers it routes to; delivery records and their claims; and notices. A click on a card whose evidence changed decides nothing. Results of authorized work are outcomes, never items. Without the resident service, Home says "Notifications stop when this window closes".

The CLI twins are `crewhaus inbox` and `crewhaus notify status|test|digest --preview`. Without Home, the CLI folds directly.

## Dependency Notes

Depends on `@crewhaus/control-contract`, `@crewhaus/control-records`, `@crewhaus/control-service`, `@crewhaus/access-control`, `@crewhaus/session-store`, `@crewhaus/feedback-distill`, `@crewhaus/harness-advice`, `@crewhaus/checkpoint-store` and `@crewhaus/deployment-controller`. Three seams are planned on existing packages: `permission-engine`'s `./authority`; `tool-notify`'s `./digest`, `./quiet`, `./sign` and `./smtp`; and `harness-advice`'s `./advisor` and `./failure-clusters`, where the Advisor's derivation and failure clustering move first.

It never enters a bundle (a guard test) and never imports `hangar-server`. It runs in Home, the CLI and the resident service, and no compiled harness reads it. No new third-party dependency.

## First Implementation Slice

`feat(attention): one inbox for every decision and problem across harnesses` adds the package, read-only, with its v1 `attention.*` operations. The notifications PR then adds `./notifications`, and later PRs extend it with acting on items and the delegation kinds. This brief lands in crewhaus/docs before the package does.

## Study References

`packages/hangar-server/src/advisor.ts` (the Advisor whose ids move to evidence keys); `packages/hangar-server/src/notifications.ts` (the console's in-memory rules this replaces); `packages/target-channel-bot/src/index.ts` (the generated bot's approval cards); `packages/tool-notify/src/lib/quiet.ts` (`quietDecision`, correct across daylight saving); and [#517](https://github.com/crewhaus/factory/issues/517).

## Validation Plan

Catalog tests: T1, T3, T8. Primary risks: **a read that writes**: every adapter is driven against a fixture fleet and every file stays byte-identical; **a wrong merge hiding a failure**: a test holds that message similarity never merges across harnesses; **a dismissal hiding later evidence**: an old dismissal hides only older evidence; **content in a notice**: a test plants a canary in tool input and asserts it reaches no email, OS or webhook payload, and reaches a card only masked and only for a member who reads content; **two deciders**: a person and a fallback both go through the one decision claim.

Residual, named rather than closed: a notice is deduplicated, not guaranteed unique, because a retry after an uncertain send can redeliver.

Definition of done: `bun test packages/attention/src` green; every head computes the same id for the same evidence; a harness's items answer the same from Home and from the CLI with Home stopped; and a guard test keeps the package out of every bundle.
