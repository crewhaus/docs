# tool-work

Status: planned
Dependency phase: 11 - Built-in tool implementations
Catalog layer: R4 - Built-in Tool Implementations
Origin in ordering: 0.9.0, the Home release ([#516](https://github.com/crewhaus/factory/issues/516))
Workspace home: packages/tool-work
Targets: the shapes the generated support table marks for work items and workspaces
Test layers: T1, T3, T8

## Purpose

What a harness was asked to do, and the path from an accepted request to one evidenced pull request. A work item holds a request's goal, acceptance criteria, source and runs. The publish engine changes a repository from an isolated checkout, checks the result, and records each step before it happens. Home, the 0.9.0 name for Hangar's web console, shows the same items in its Work section.

It is the work store, the Work tools and the one package that talks to a git remote with credentials. `tool-git` keeps remotes out by design and stays that way. The runtime decides where and how a change is published; the model decides what. Issue, task or agent text can propose work, and only a person or the owner's policy accepts it.

## Boundaries

Owns:

- **The tools.** `WorkList`, `WorkGet`, `WorkPropose`, `WorkUpdate`, `WorkspaceOpen`, `WorkspaceCheck` and `WorkspacePublish`, wired by `scripts/wire-tool-package.ts`, and the `tool_config.work` registrar. A spec gets them only by listing them, and a bundle carries them through the builtin table. They propose, add notes, link runs and close with a reason; they never accept, decline or cancel, never rewrite an accepted item's goal or criteria, and never claim an operator's or API caller's source.
- **The work store (`./work-items`).** A content-free append-only log under `.crewhaus/run/control/work/`; text in immutable content revisions (mode 0600), each field recording the origin it came through; and source claims under a keyed digest, so duplicate events update one item. Items are `wi_<ulid>` and move proposed, open, closed, and reopen.
- **One proposal object.** A proposed item carries its assessment: problem and audience, dated evidence labelled observed, reported or hypothesis, a basis, fit with the harness's purpose, alternatives and, for a feature, what happens if nothing is done. A simulated reaction is never a request.
- **One ranker.** Priority, then the owner's weights in `tool_config.work.ranking`, then age, each dimension shown, never one score posing as a probability. A person corrects the order with an example whose scope is recorded.
- **The publish engine (`./workspace`).** One managed bare clone per binding at `.crewhaus/workspaces/<binding>/repo/`, one worktree per accepted item on a branch minted once, and every git call through `tool-git`'s `./run` with hooks off. Argv comes from fixed verbs, minted branch names, full SHAs and the connection's clone URL; no request body or model text becomes argv.
- **Checks and revalidation.** `WorkspaceCheck` runs only the declared checks, never through a shell, in the configured sandbox, or on this computer only after the owner confirmed. Before publishing, a moved base is rebased and checked again, and a conflict refuses with `conflicts_with_base`.
- **The acceptance check.** Before every publish a judge reads the change against the item's criteria and, when the spec declares one, `intent:`. `intent:` is the judge's input only: no prompt and no tool-work result carries it or reasons that could quote it. The verdict and reasons go on the publish attempt, the pull request's body and Home's card, and the agent gets the verdict without reasons that could quote it.
- **Idempotent publish and reconcile.** Each attempt holds a claim and logs `started`, `pushed`, then `done`, `failed` or `unknown`. A pull request is created or updated by its branch, so an answer lost to a crash is reconciled against the host, never a second pull request.
- **The one PR outcome tracker.** A control-plane sweep records merged, merged with changes, closed unmerged or reverted. Items close from it, and `tool-project` reads it rather than recomputing it.
- **Credential custody for git.** A provider supplies the token. It reaches only the git process making that network call, through that process's environment, as an extra header for the codehost's origin; never argv, a URL, `.git/config` or disk. Clone, fetch and push share this one mechanism.
- **Its classifying sites.** Each text is classified once, at the origin it arrived through, field by field, before a model sees it, inside `WorkList` and `WorkGet` and in the acceptance check: `api` for what a member or API caller wrote, `memory` for what the harness wrote in an earlier run, `mcp` for an adopted Thredz task, and `source` for adopted issue text. The read tools set `classifyOutput: false`, so the same text is not classified again at `tool`.

Does not own: the `repositories:` and `tool_config.work` keys ([#514](https://github.com/crewhaus/factory/issues/514)); repository connections, bounded discovery and the project record (`tool-project`, brief 336); the lease and the App that mints it (`github-app`, brief 338); the rules that may delegate a merge and the sweep that applies them (`control-service`, brief 325); inbox items (`attention`, brief 331); the `work.*` operations as data (`control-contract`, brief 322); and the budget ledger (`control-records`, brief 324).

## Delegated merge and revert

Status: planned

Off until an owner turns a rule on, and it adds no package. Merges happen only under a rule the owner turned on, after held-out checks, with an automatic revert. **No tool merges:** this package's manifest has no merge tool, and a parked `gh pr merge` through `Bash` still asks a person. A person merges from Home or with `crewhaus work merge|revert`, through `work.merge` and `work.revert`, which resolve the same decision and run the same executor.

- **The engine.** `mergePullRequest`, `runHeldOutChecks` and `openRevert` join `./workspace`.
- **The gates.** Five, each for the exact head, in [#516](https://github.com/crewhaus/factory/issues/516): scope, host, acceptance, held out, one at a time. The host gate reads the head's checks fail-closed through `rollupConclusion`, and held-out checks come from the active deployment, never the pull request.
- **The effect.** It records its rollback plan, re-reads base and head, and merges with GitHub's `sha` parameter set to the decided head, through the publish engine's credential provider. A moved head is refused and supersedes the decision. A required review refuses it, because CrewHaus never changes branch protection.
- **The watch.** A failing check on the merge commit on the base opens a revert pull request whose diff is the merge's exact inverse, merged under the rule's rollback clause once the host gate passes. The rule is suspended and one urgent item says why. A revert that cannot merge goes to a person and is never retried blindly.
- **A new edge.** This section's PR adds `deployment-controller` to the dependencies, for `rollupConclusion` through its planned `./approval-gate`.

It lands with `feat(tool-work): merge a harness's own pull requests under its owner's rule, after held-out checks, with an automatic revert`.

## Inputs and Outputs

Inputs: tool calls in a harness; `crewhaus work` verbs and `work.*` operations from Home and the API; `runs.create` with a `work` field; the lowered `repositories:` binding and `tool_config.work`; a token from the credential provider; and pull-request state from the host, which is enums and ids and reaches no model.

Outputs: work items and revisions, worktrees and branches, check logs, publish attempts, one pull request per item whose body links the request, plan, decisions, tests, verdict and open concerns, and content-free `work.*` events. A run with an item resumes in the scope `work:<item>`, so a grant decided in the inbox reaches the run that picks the item up again.

## Dependency Notes

Depends on `@crewhaus/tool-builder`, `@crewhaus/tool-catalog`, `@crewhaus/tool-safety`, `@crewhaus/control-contract`, `@crewhaus/control-records`, `@crewhaus/continuity-store`, `@crewhaus/durable-execution`, `@crewhaus/egress-classifier`, `@crewhaus/tool-git`'s `./run`, which is on main, and `@crewhaus/tool-codehost`'s planned `./pulls` and `./credentials`. It is light, and adds no third-party dependency. A tool that crosses to the host or runs project code sets `scope: "external"` and its I/O capability. A spec that declares no new key compiles to the same bytes.

## First Implementation Slice

`feat(tool-work): work items and the Work view, fed by goals, plans, Thredz tasks, jobs and runs` adds the package with the store and the four Work tools. Later PRs extend it, first `feat(tool-work): deliver work items as pull requests from isolated, re-validated checkouts`, then the section above. This brief lands in crewhaus/docs before the package does.

## Study References

`packages/tool-git/src/index.ts` (the no-remote boundary this completes); `packages/tool-codehost/src/index.ts` (`PrCreate`, which needs a branch someone already pushed); `packages/tool-deploy/src/index.ts` (why no promote tool exists: a tool cannot see how its call was authorized); `packages/continuity-store` (goals and plans the Work feed joins); and [#516](https://github.com/crewhaus/factory/issues/516).

## Validation Plan

Catalog tests: T1, T3, T8. Primary risks: **a second pull request after a crash**: a journey kills the engine after the host answered and asserts one pull request; **a credential at rest**: a test scans argv, `.git/config` and the clone after a push; **text granting authority**: a planted instruction in an issue cannot accept an item or change a destination; **a merge that outruns its evidence**: a seeded failing held-out check refuses it, a moved head supersedes it, and a seeded failure after a merge reverts it.

Definition of done: `bun test packages/tool-work/src` green; a loopback journey delivers a fix as one pull request; `crewhaus doctor --philosophy-alignment` exits 0 with every site listed; and every landing PR asserts zero byte-pin diffs.
