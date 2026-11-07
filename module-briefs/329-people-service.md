# people-service

Status: planned
Dependency phase: 18 - Deployment & studio
Catalog layer: F3 - Deployment & Operations
Origin in ordering: 0.9.0, the Home release ([#521](https://github.com/crewhaus/factory/issues/521))
Workspace home: packages/people-service
Targets: All (it operates compiled harnesses; it is not embedded in one)
Test layers: T1, T3, T8

## Purpose

Profile management for Home, the 0.9.0 name for Hangar's web console, and for the `crewhaus people` verbs. A person creates a profile, reviews what it says, lets a harness use part of it, previews the difference, exports it, stops a harness using it and erases it. This package holds each of those acts; brief 328 holds what a harness loads.

It stays apart from `@crewhaus/people` for two reasons. It depends on `control-service` and every harness store, so it must stay out of bundle trees: a guard test fails if any emitter or codegen twin renders it. And it is a separate authorization domain: no workspace role reaches a profile, so a harness owner can neither read one nor approve its erasure.

## Boundaries

Owns:

- **Persons and notes.** Three kinds: personal (`personal`, the owner or someone they look after, with a recorded basis), group (`audience`, never facts about one member) and made-up persona (`synthetic`, labelled as made up wherever it is used). A note (`claim` in the API) carries its situation, source, how it was learned, confidence, sensitivity, a review date and the purposes it may serve. There is no completeness score.
- **The rules in code.** Health, beliefs, sexuality, ethnicity and biometrics are only ever stated by the owner, never inferred. Sensitive notes become active only by the owner's action. A profile about someone else holds such notes, or learns from conversations, only with that person's recorded consent. A rejected note's keyed fingerprint keeps it out, from every source, until reopened, and the owner's correction wins over inference.
- **Review and collection.** The review tray, automatic updates that are opt-in for ordinary areas and come with undo, and conflicts that are never settled silently. `crewhaus people review --collect`, or the resident service, brings in suggestions harnesses sealed.
- **Interviews, imports and reviewed learning.** A short interview of skippable questions; imports of documents through `@crewhaus/tool-docs`' readers, conversations, work samples, exports and old preference files, each suggestion with its excerpt and source; reactions to pairs of answers.
- **Uses and packs.** A use (`grant` in the API) lets one harness use one person's notes through one slot, with purposes, areas, where answers may go, who may process them and an optional end. Only the owner or custodian grants one, after seeing the harness's data path, and "Only this computer" is offered only when every step is local. Stopping a use deletes its packs and withdraws any approval waiting in a conversation that used it (`people-grant-revoked`).
- **Preview and estimates.** A harness's answer with and without the profile, and "What would they choose?". Each runs as a side call with tools off and `persistSession: false`, only on a service the person approved for that step.
- **Export.** Versioned `crewhaus.people.v1`, sources and sensitive notes off by default, rejected notes included so they stay rejected. The preview's fingerprint matches the downloaded file. An imported export is untrusted: it lands in review, refuses instruction-bearing fields and drops any use or permission it carries.
- **Erasure, the one sanctioned hard-delete path for a person's data.** The owner or custodian decides, where the basis allows; the workspace owner only for a removed member's profiles, through an inbox item that shows a count, never names. No harness owner approves it, and no retention pin holds the records back. A dry run comes first and lists what will go and what cannot, with dates or expiry. A typed name then runs idempotent steps, each resumed after a crash: revoke the uses, cancel runs in those conversations, supersede their pending decisions (`people-erased`), delete the person's key and records through the vault's crypto-shred, rotate the space key, purge each harness's derived records through `knowledge-index`'s `./erasure`, and rotate each affected harness's digest key. The receipt holds no content.
- **The people-domain checker.** Every `people.*` operation is decided here, through permissions that never enter the role matrix ([#507](https://github.com/crewhaus/factory/issues/507) reserves the domain). Refusals answer 404, so nobody learns that a profile exists. A viewer the owner names reads; a subject who is a member reads the conversations about them; a service principal acts only in a custodial space bound to it.
- **Read models.** People in Home lists only the profiles a member owns, looks after or was shown. Setup › People shows a harness's slots and data path, and how many people let it use their profiles, never who.
- **Its classifying sites.** At TrustOrigin `people`, with the detector's local layers only: a note before it is saved, an import per extracted chunk, each extracted suggestion, and interview answers before a follow-up call. At `api`: a preview prompt or estimate question a member types, before the side call. [#521](https://github.com/crewhaus/factory/issues/521) gives each site's verdicts.

Does not own: custody, sealing, packs on disk, selection, rendering and the reply screen (`people`, brief 328); the store purges and the refusal of ordinary knowledge (`knowledge-index`'s `./erasure`, brief 335); the per-call fence and data-path manifest (`model-service`, brief 308); commands, receipts and the erasure hooks it calls (`control-service`, brief 325); the wire operations as data (`control-contract`, brief 322); the HTTP surface (`hangar-server`, brief 301).

## Inputs and Outputs

Inputs: an authenticated member or the local actor; an operation and its fields, each text field declared with [#513](https://github.com/crewhaus/factory/issues/513)'s `modelBound` rule; the vault; a harness's lowered `people:` block, for its slots and data path; and uploaded files, as untrusted bytes.

Outputs: sealed records and packs through `./vault`; a content-free workspace receipt for each change; review items; a preview, an estimate and an export file with its fingerprint; erasure's dry-run list and final receipt, whose only claim is that a record of the erasure is kept and contains nothing from the profile. A note's value never goes on a command line, and every action has a `crewhaus people` twin or `null` with the reason.

## Dependency Notes

Depends on `@crewhaus/people`, `@crewhaus/access-control`, `@crewhaus/control-service`, `@crewhaus/knowledge-index` (`./erasure`), `@crewhaus/session-store`, `@crewhaus/memory-store`, `@crewhaus/wiki-store`, `@crewhaus/continuity-store`, `@crewhaus/data-retention-engine` and `@crewhaus/tool-docs`. It registers through the area surface of [#513](https://github.com/crewhaus/factory/issues/513), so control-service never imports it, and it never imports hangar-server. No new third-party dependency.

Home's assistant never reads items marked `people`, and profiles stay out of the knowledge index and search.

## First Implementation Slice

`feat(people): people profiles live in an encrypted vault outside every harness` adds the package with the admin v1 and `crewhaus people`, beside brief 328's `./vault`. Later PRs extend it: `feat(people-service): erasing a person removes their data everywhere, and people data stays out of shared knowledge`, which lands before any harness can use or import a profile, and `feat(people-service): profiles are built by interview, import and reviewed learning`. This brief lands in crewhaus/docs before the package does.

## Study References

`packages/harness-lifecycle/src/retention.ts` and `packages/data-retention-engine/src/retention-config.ts` (what keeps records, and so what erasure must override for a person's records alone); `packages/feedback-distill` and brief 300 (de-identifying text for learning, the practice a vault deliberately departs from); `packages/hangar-server/src/m3.ts` (the write covenant [#520](https://github.com/crewhaus/factory/issues/520) and [#521](https://github.com/crewhaus/factory/issues/521) rewrite once, to name every sanctioned path that deletes); and [#521](https://github.com/crewhaus/factory/issues/521).

## Validation Plan

Catalog tests: T1, T3, T8. Primary risks: **someone learning a profile exists**: refusals answer 404, and every read model is tested for what a viewer, a custodian and a workspace owner each see; **an erasure that stops halfway**: each step is idempotent and a test crashes between every pair; **an erasure that claims too much**: the dry run names what it cannot reach, such as downloaded exports, earlier backups and messages already sent; **instructions planted in an import**: a malicious chunk is left out and listed with the reason, and an imported export's uses and permissions are dropped.

Residuals, named rather than closed: copies a model provider keeps under its own terms, and shared notes that do not record which conversation wrote them, both listed by the dry run.

Definition of done: `bun test packages/people-service/src` green; the guard test keeps it out of every bundle and every bundle's install tree; after an erasure, audit `verify()` still passes and the erased text cannot be read; a fake-model journey runs a whole profile from creation to erasure; and `crewhaus doctor --philosophy-alignment` exits 0 with each site listed.
