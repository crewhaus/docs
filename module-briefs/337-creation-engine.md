# creation-engine

Status: planned
Dependency phase: 18 - Deployment & studio
Catalog layer: F4 - Studio & Authoring UX
Origin in ordering: 0.9.0, the Home release ([#515](https://github.com/crewhaus/factory/issues/515))
Workspace home: packages/creation-engine
Targets: All (it operates compiled harnesses; it is not embedded in one)
Test layers: T1, T3

## Purpose

One way to create a harness. A newcomer in Home, the 0.9.0 name for Hangar's web console, says what they want help with, and the engine reads the project, reflects what it understood, asks only questions that change a setting, and generates a spec they can rehearse before anything runs. `crewhaus init` uses the same engine and keeps its output.

`crewhaus init` and Home's guided creation must share one engine, and neither can host it: `apps/cli`'s only export runs the argv switch on import, and the CLI imports only the start surface of `hangar-server`. So it is a package of its own, which never enters a bundle.

## Boundaries

Owns:

- **The engine (`.`).** The outcome catalog, with Hangar's `WIZARD_TEMPLATES` folded in and one current default model; the credential-free questionnaire; `init`'s interview and scaffold, moved with no behaviour change; deterministic project claims; reflection; the provenance check; question selection; the two-way agreement view and its consequence text; and generation of spec edits, eval assets and the deployment plan.
- **Outcomes.** Home asks "What would you like help with?": maintain a project, research a topic, help customers, manage recurring work, or start from something else. The shape stays a technical detail, and a catalog test holds that it can do what the outcome promises.
- **Reflection.** One model call with no tools, within a per-session spend cap, through Home's model connection. Its claims and questions are validated and kept, so a replay is exact. Each line is marked "You told me", "The project says" or "I inferred", and "The project says" stands only if the quoted words appear in a classified excerpt. A question that would change no setting is never asked. Without a model, the outcome's fixed questionnaire runs. An unpriced model reads "Cost unknown", never `$0`.
- **The agreement.** A preset and plain rows, each with a control, a consequence line and its spec paths under technical details. The footer reads "Saved as a draft. Nothing runs yet."
- **Authority from choices alone.** Permission rules, action classes, repositories, scopes, budgets, recipients and connections are written only from choices a person made. Reflection produces claims, never settings, and instructions come from confirmed claims through templates. Generation grants no role: when the creator is not the owner, the owner-only rows go to the owner as one decision. Creation merges and deploys nothing on its own and never sets up a delegation.
- **Durable sessions (`./session`).** A `crs_` session under the Hangar root, with a head, content-free step and resource ledgers and state machines as data. Resume runs reconcilers before it retries, so a resource created but not finished is reused, not created twice. Cancel lists what the session created and cleans up dry first, touching only those resources. `crewhaus init` leaves one already generated, so Home can offer "Finish setting up".
- **The gate evaluator (`./gates`).** A pure evaluator of the five activation gates (rehearsal, connections, authority, decision owner, spend), each passing only on evidence, registered into core activation.
- **Its classifying sites.** At TrustOrigin `api`, what a member writes into a setup session, classified in the process that makes the model call, before each reflection or interview call, and tagged; blocked text is never sent, and the member sees why. At `source`, each watch-me observation used as an interview seed. Reflection's excerpts were classified at the collector, so the engine only tags them and classifies nothing twice.

Does not own: reading a project and the project record (`tool-project`, brief 336); guarded spec writes (`spec-patch`'s planned `./commit`); checking and compiling a draft (`compile-service`, brief 334); kept builds, activation and the gate list that runs these gates (`control-service`, brief 325); rehearsal capture (`eval-runner`); the structured editor and creation screens (`hangar-ui`, `hangar-server`); the spec keys it writes ([#514](https://github.com/crewhaus/factory/issues/514)); and delegation rules (brief 325).

## Inputs and Outputs

Inputs: an outcome and a person's answers from Home, the API, `crewhaus create` or `crewhaus init`; a project record; registry templates, reached only through `verifyingRegistry`; a model connection, when there is one; and resource facts from reconcilers.

Outputs: a session, its ledgers, confirmed claims, an agreement view, and a generated harness: a spec with `intent:` and initial criteria, the rehearsal case, an examples dataset with deterministic graders first, and a deployment plan, created and registered in one command. Steps that run before a harness exists are workspace jobs whose closed list of kinds runs no agent tools.

## Dependency Notes

Depends on `@crewhaus/spec`, `@crewhaus/spec-patch`'s planned `./commit`, `@crewhaus/claim-ledger`, `@crewhaus/runtime-core`, `@crewhaus/memory-service`, `@crewhaus/tool-builder`, `@crewhaus/session-store` and `@crewhaus/template-registry`; once sessions land, `@crewhaus/control-contract`, `@crewhaus/control-records` and `@crewhaus/tool-project`; once gates land, `@crewhaus/compile-service`. No new third-party dependency. Model-backed reflection also sits behind a release switch set from a live red-team run, and it stays off, leaving the templates, if that run fails.

## First Implementation Slice

`feat(creation-engine): one creation engine, and durable creation sessions from every entry point` adds the package: the extraction from `apps/cli` and Hangar's wizard with no behaviour change, then `./session`. Later PRs extend it, first `feat(creation): build, rehearse without side effects, gate and activate`, which adds `./gates`. This brief lands in crewhaus/docs before the package does.

## Study References

`apps/cli/src/init-conversation.ts` (the interview that moves); `packages/hangar-server/src/builders.ts` (`WIZARD_TEMPLATES`); `packages/template-registry/src/index.ts` (`verifyingRegistry`); `packages/claim-ledger` (the text fold behind the provenance check); and [#515](https://github.com/crewhaus/factory/issues/515).

## Validation Plan

Catalog tests: T1, T3. Primary risks: **setup text becoming authority**: a red-team suite drives a fake model that obeys planted instructions and asserts that no authority field changes; **a repeated resource after a crash**: a test crashes between each step and asserts that resume reuses what exists; **`init` drifting in the move**: `crewhaus init` writes the same bytes for every template as before; **a gate passing without evidence**: a table test holds each gate failing on absent evidence.

Definition of done: `bun test packages/creation-engine/src` green; `crewhaus init` byte-identical per template; every outcome's catalog entry yields a shape that does what it promises; a loopback journey creates a codebase steward whose rehearsal writes nothing to the host; and a guard test keeps the package out of every bundle.
