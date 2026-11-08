# quality-service

Status: planned
Dependency phase: 9 - Telemetry & eval
Catalog layer: R15 - Telemetry, Tracing, Eval
Origin in ordering: 0.9.0, the Home release ([#519](https://github.com/crewhaus/factory/issues/519))
Workspace home: packages/quality-service
Targets: All (it operates compiled harnesses; it is not embedded in one)
Test layers: T1, T3, T5, T8

## Purpose

Carries every improvement to a decision. An optimizer result, a flywheel result, an advice item, a clarification or a guided repair becomes a change record with its evidence, a held-out check and a decision someone with the authority takes from any surface. It also answers "how do we know it is good?" for each harness's Quality section in Home, the 0.9.0 name for Hangar's web console.

It is a package of its own because the improvement lifecycle and the quality read models sit above the eval stack on control records. The eval libraries measure rather than govern, and control-service is where areas register, not what they extend. It links the proposal, approval, canary and rollback machinery that exists rather than replacing it, and passing evals never grant authority.

## Boundaries

Owns:

- **Change records.** Content-free `chg_` rows in `.crewhaus/run/control/changes.jsonl`, on `control-records`' primitives, with the diff and evidence in `.crewhaus/proposals/<chg>/` (mode 0600), where `crewhaus propose` already writes. A row names its source from a closed list, its base (spec, graders, criteria, dataset version, deployment), a keyed patch digest, the surfaces it touches as the spec editor classifies them, and whether it touches a protected criterion or bar, or `intent`. A change runs from proposed to kept, or ends rolled back, declined, withdrawn or superseded. The rows sit behind [#507](https://github.com/crewhaus/factory/issues/507)'s runtime-state floor.
- **The held-out promotion check.** A graders change regrades the base run's recorded answers with no agent call. A dataset change runs only the added cases. A spec change runs an isolated eval with tool results replayed from the base run's recording, then base and changed spec on the holdout, one counted use each, gated strictly. The evidence keeps both runs, intervals, cost and the holdout result; the paired significance eval-report computes is decision support, never the gate. Without a holdout, a person must acknowledge "Adopt without a held-out check".
- **Adoption.** A passing check opens a "Ready to decide" `dec_` decision through the decision service, bound to the change, its patch digest, base and evidence: a maintainer's, or the owner's when it touches a protected criterion or bar, or `intent`. Adopting checks again at that moment, writes through the sanctioned writers and stops at "Adopted, not active yet". Activation stays the deployment activation of brief 325, which this package observes.
- **Regrades.** Orchestrating `crewhaus eval --regrade` and previews over recorded answers, only with graders whose inputs were recorded; the rest report `not_regradable`.
- **The quality read model.** The per-harness model behind the Criteria, Cases, Graders, Results and Improvements tabs, the criteria fold and revision history, and the fleet fold under a disposable Hangar-root projection.
- **Guided repair.** Failing, not-measured and ambiguous cases grouped by criterion, cause and theme without a model. An ambiguity becomes a question ("Which answer is right for case csv-017?"), asked again only with new evidence, and its answer becomes a proposal. Clarifications are templates by default; a model draft is opt-in, metered, classified and labelled "Drafted". The attention edge arrives with the guided-repair PR.
- **The watch.** Activation opens a post-activation watch, 24 hours by default. A worse comparable eval, a failing drift-sentinel run, or in-loop failures and not-measured turns raise an item with Roll back and Keep. Rolling back applies the inverse change, refused with the diff if the spec moved since, and rolls the deployment back.
- **Its classifying sites.** The stored answers, judge reasons and feedback a draft is built from, each at `record` before the drafting call; the draft itself at `api` before it is shown; and annotation, correction and preference notes at `api` when written, so a malicious note is refused and nothing is stored. Each site joins doctor's boundary list in the PR that adds it.

Does not own: the holdout use ledger, `<version>.uses.jsonl` beside the dataset version (`dataset-registry`); grading, result states and comparability (`eval-grader`, `eval-report`, `eval-ops`, extended in place); protected bars and the checks that refuse loosening them (`spec-patch`); the inbox, items and decision lifecycle (`attention`, brief 331); deployments and activation (`control-service`, brief 325); the rollout action a delegated adoption hands its build to (brief 325's slot); the wire operations (`control-contract`, brief 322); and the Quality screens (`hangar-ui`).

## Delegation (`./delegation`)

Status: planned

Off until an owner turns a rule on, and it adds no package. Adoption asks a person unless the owner has turned on a narrow rule for routine changes, inside a ceiling read only from the active deployment. The rule and its consequences page are brief 325's; this entry defines what is routine and what adopting under a rule does.

`routineChangeOf` is deterministic and names the first condition that fails, in [#519](https://github.com/crewhaus/factory/issues/519)'s order: `source_not_routine`, `touches_protected`, `surface_not_routine`, `security_surface`, `human_owned_path`, `loosens_check`, `dataset_not_additive`, `not_checked`. So a regression case can be routine, while a clarification that accepts more never is.

The `adopt` action resolves the decision only if the comparison passed with no regressions and a held-out pass, and the change is still fresh. The control plane adopts as a system actor applying the owner's rule, builds, keeps and rehearses the result, and hands the rollout decision to the same rule. The `rollout` action itself is in brief 325's slot. If the rule ends in between, the change reads "Adopted automatically; activating it needs you", and a person activates it. Passing evals alone never grant.

A regression during the watch rolls back to the previous kept build, applies the inverse change, suspends the rule, which only the owner can resume, and raises one urgent item. A rule never decides anything owner-class because of `govern`, and while neither Home nor `crewhaus delegations apply` is running, the decision waits for a person.

It lands with `feat(quality-service): routine improvements and saved revisions roll out under an owner's rule, watched, with an automatic rollback`.

## Inputs and Outputs

Inputs: a harness directory; a proposal from a closed list of sources (Home's "Propose this", `crewhaus changes propose`, `crewhaus propose`, guided repair, criteria edits); eval run artifacts and the run index; a held-out split read through the promotion check; an authenticated actor; and the deployment records it watches.

Outputs: `chg_` rows, evidence directories, `dec_` decisions through the decision service, receipts for each mutating operation, content-free `change.*` events on the harness stream, and read models whose every state has a word and a reason. An unknown is never a pass or a fail.

## Dependency Notes

Depends on `@crewhaus/control-contract`, `@crewhaus/control-records`, `@crewhaus/eval-report`, `@crewhaus/eval-ops`, `@crewhaus/eval-grader`, `@crewhaus/dataset-registry`, `@crewhaus/spec-patch`, `@crewhaus/spec-changelog` and `@crewhaus/feedback-distill`, and on `@crewhaus/attention` from the guided-repair PR. It registers through the area surface of [#513](https://github.com/crewhaus/factory/issues/513), never imports hangar-server, and never enters a bundle (a guard test). No new third-party dependency, and no new permission: reading answers needs `h.read.content`, running evals `h.eval.run`, editing `h.quality.edit`, and anything that loosens a protection the owner's `h.spec.edit.security`.

## First Implementation Slice

`feat(quality-service): held-out checks and change records on the governance lifecycle` adds the package. Later PRs extend it, first `feat(quality-service): improvements and guided quality repair`, then the delegation section's PR. The eval and grader libraries' own changes land in their own PRs before this one. This brief lands in crewhaus/docs before the package does.

## Study References

`apps/cli/src/propose.ts` (where a proposal's bundle is written); `packages/eval-optimizer-orchestrator/src/failure-arbiter.ts` (the rules guided repair feeds); `packages/dataset-ops/src/graders-suggest.ts` (model-free clustering); `packages/tool-evalops/src/lib/lineage.ts` (cutting a line where the instrument changed); and [#519](https://github.com/crewhaus/factory/issues/519).

## Validation Plan

Catalog tests: T1, T3, T5, T8. Primary risks: **an improvement adopted on evidence that no longer holds**: adopting checks base, instrument and authority again, and otherwise reads "Out of date"; **a held-out case spent without a record**: only the promotion check and a release read the holdout, and each read counts; **a bar lowered by a routine path**: a table test covers every `routineChangeOf` reason, and an owner-class change never resolves under a rule; **a screen that reads what was never written**: screens read real eval artifacts produced in the test.

Definition of done: `bun test packages/quality-service/src` green; a browser journey against loopback fakes takes a failing case through guided repair to a watched change, and adopts and rolls back a routine repair under a rule; `crewhaus doctor --philosophy-alignment` exits 0 with the three sites listed; and a guard test keeps the package out of every bundle.
