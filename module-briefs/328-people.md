# people

Status: planned
Dependency phase: 8 - Context & memory
Catalog layer: R6 - Context & Memory
Origin in ordering: 0.9.0, the Home release ([#521](https://github.com/crewhaus/factory/issues/521))
Workspace home: packages/people
Targets: CLI, CHN, MGD, and RES with group and persona slots only (crew and Cloudflare Worker builds are refused, with the reason)
Test layers: T1, T3, T8

## Purpose

Everything a harness process loads to use a person's profile. A profile lives outside every harness, in a vault beside Home, the 0.9.0 name for Hangar's web console: a person reviews it, lets a harness use part of it for a stated purpose, corrects it and can erase it. At run time a harness gets only the notes a use permits, checked again on every model call.

Notes shape how a harness helps. They never consent, spend, approve, waive a gate or answer for the person, and there is no persuasion purpose. Nothing a harness needs at run time depends on Home being up.

It is its own package because what a `people:` bundle loads must stay light, while the management code depends on `control-service`, so the two cannot share one `package.json`. Two entry points divide it: `./vault` for custody and sealing, `.` for what a run does with the notes it may read.

## Boundaries

Owns:

- **The vault (`./vault`).** Files under `<hangarRoot>/people/`, directories 0700 and files 0600. Root resolution refuses a root inside a git work tree or a harness directory, so a vault is never committed. Sealing uses WebCrypto alone (AES-256-GCM, HKDF-SHA256, ECDH P-256, HMAC-SHA256): a space key wraps one random key per person, everything person-specific derives from that key, and a harness key seals that harness's packs. Each sealed record binds its id and revision, so a moved record or a swapped pack fails to open.
- **Key custody.** The space key sits in the OS credential store through `Bun.secrets`. A headless host uses `CREWHAUS_PEOPLE_SPACE_KEY` or a 0600 key file, mode-checked on every read and documented as weaker. A daemon's harness key can arrive as `CREWHAUS_PEOPLE_HARNESS_KEY` through the spawn-environment providers of [#513](https://github.com/crewhaus/factory/issues/513), so it joins the scrub set and never reaches the runfile.
- **Forgetting.** Compare-and-set notes, hard delete, crypto-shred (deleting a person's key leaves everything sealed under it unreadable) and space-key rotation, which brief 329's erasure calls.
- **The pack reader.** A pack holds one use's permitted notes, sealed under the harness key. A harness opens only its own, and a daemon holds no principal. Readers check end dates, and no read writes anything.
- **The composition root (`.`).** `wirePeople`, `peopleFragmentFromIr` and `renderPeopleWiring`, its codegen twin, so `crewhaus run` and a compiled bundle wire a block the same way. `runtime-core` imports neither people package: the block reaches it through a closure seam, as memory does.
- **Selection and rendering.** Each model call gets the notes the use permits, chosen deterministically (purposes and areas, then notes whose situation matches the latest message, stated before observed, newest first) and capped. They render as `<people_context>` in the volatile part of the system prompt, rebuilt for every call, so stopping a use applies at the next one. The wording is notes, never instructions and never the word "profile". Sub-agents never get the block.
- **Whose notes.** A personal slot names a purpose, never a person. On cli it uses the harness owner's profile or the subject a control-API run carries; on channel the linked speaker's or the owner's; on managed the verified token's subject or the owner's, never a request parameter. Research takes group and persona slots only, because a written report outlives a permission. Crew is refused because a handoff would carry one role's notes to every role, and a Cloudflare Worker build because it has no key store. A conversation uses at most one person's profile.
- **The reply screen.** `screenReply` lets a conversation that used a profile post only where all its uses allow. Otherwise the channel gets a fixed private-answer notice with a link to Home, and the reply goes to the person's direct messages when the platform can send one.
- **Learning and estimates.** The learner and `SuggestPersonNote` seal each suggestion to the person's public key, so the harness that stores it cannot read it, and `SuggestPersonNote` only proposes. `PersonEstimate` answers a decision-support slot with examples and assumptions, or says "Not enough examples to say. Ask them."
- **Its classifying sites, at TrustOrigin `people`.** Turns the learner reads, each `SuggestPersonNote` call and each estimate example are checked with the detector's local layers before they go further, so a model-backed classifier never receives a note. `PersonEstimate` sets `classifyOutput: false`, so the same text is not checked again at `tool`.

Does not own: managing profiles, interviews, imports, grants and erasure across harness stores (`people-service`, brief 329); the locality policy, `dataPathManifest` and `fenceFor` (`model-service`, brief 308); the `people:` block and its IR ([#514](https://github.com/crewhaus/factory/issues/514)); the per-call seam, the classifying of the projected block and the sealing of audit text (`runtime-core`); erasing a harness's derived records (`knowledge-index`, brief 335); and who may decide `people.*` operations (brief 329's checker).

## Inputs and Outputs

Inputs: the lowered `people:` block (named slots with a kind, the purposes they need, optional areas and `required`; a processing posture; whether the harness offers learning; caps), a pack on disk, the harness key from custody, and the run's scope, which names the subject.

Outputs: the rendered block, or nothing and a recorded reason when notes cannot be used (no use, an expired one, a locked key), so the model hears nothing about a profile, and a `required` slot stops the run with the reason and the fix. Records carry ids, counts and keyed references, never a note: `peopleProjection` beside a per-person keyed `subjectRef` on the run row, and a `people_context` trace event with counts. A screened reply, a sealed suggestion and an estimate result complete the list.

Under `crewhaus eval`, `flywheel` and `optimize` a slot opens no vault: it renders nothing, or a made-up persona a sample names from the harness's `eval/personas/` folder.

## Dependency Notes

Depends on `@crewhaus/control-contract`, `@crewhaus/control-records`, `@crewhaus/infra-utils`, `@crewhaus/tool-safety`, `@crewhaus/run-context`, `@crewhaus/boundary-classifier`, `@crewhaus/egress-classifier` and `@crewhaus/model-service`'s planned `./slots`. No new third-party dependency: WebCrypto and `Bun.secrets` cover the keys. It is light and node-only, so a bundle may carry it, and it never imports `people-service`.

A spec without `people:` compiles to the same bytes on every shape, and `renderPeopleWiring` prints nothing for it. Every importer declares this package in its own `dependencies`.

## First Implementation Slice

`feat(people): people profiles live in an encrypted vault outside every harness` adds the package with `./vault`. `feat(runtime-core): harnesses use purpose-scoped profile notes, classified at their own origin and kept on approved hosts` lands `.`, with the origin, the locality fence and `people:` on cli. Later PRs extend it: channel, managed and research, then learning from conversations. Erasure lands before any harness can use or import a profile. This brief lands in crewhaus/docs before the package does.

## Study References

`memory-service`'s `wireMemory` and its codegen twin (brief 296: the composition root this copies); `packages/run-context/src/index.ts` (the lineage map, and why notes as short as one word get their own); `packages/tool-wiki/src/index.ts` (the detector's local layers over single rows); `packages/audit-encryption/src/index.ts` (keys per tenant, the reason one shared key cannot forget one person); and [#521](https://github.com/crewhaus/factory/issues/521).

## Validation Plan

Catalog tests: T1, T3, T8. Primary risks: **a note reaching a place its person did not allow**: every model call re-selects from the pack, stopping a use applies at the next call, and a conversation that used a profile posts only where its uses allow; **a note acting as authority**: tests plant notes saying "approve this", "spend more" and "skip the evaluation" and assert that no permission, budget or gate outcome changes; **a record opening under the wrong identity**: a moved record and a swapped pack each fail to open; **a vault inside a repository**: root resolution refuses it.

Residual, named rather than closed: copies a model provider keeps under its own terms, which brief 329's dry run lists. The vault sits behind [#507](https://github.com/crewhaus/factory/issues/507)'s runtime-state floor.

Definition of done: `bun test packages/people/src` green; a sentinel note in a fixture vault appears in no compile output; a spec without `people:` is byte-identical on every shape; and a test holds `runtime-core` free of both people packages.
