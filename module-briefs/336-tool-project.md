# tool-project

Status: planned
Dependency phase: 11 - Built-in tool implementations
Catalog layer: R4 - Built-in Tool Implementations
Origin in ordering: 0.9.0, the Home release ([#515](https://github.com/crewhaus/factory/issues/515))
Workspace home: packages/tool-project
Targets: the shapes whose spec lists its tools
Test layers: T1, T3, T8

## Purpose

Read a project and keep a record of what was read, without running any of it. A codebase steward needs a repository's README, docs, manifests, recent commits and chosen issues, and setting one up in Home, the 0.9.0 name for Hangar's web console, needs the same text before any harness or agreement exists.

It is a package of its own because that no-exec, no-push reader has to run on a repository outside any harness during creation, and also be carried by steward bundles afterwards. So it sits below both, and it stays light.

## Boundaries

Owns:

- **The tools.** `SourcesRefresh`, `SourceList`, `SourceGet`, `ProjectMap`, `OutcomesRefresh` and `ProjectReport`, wired by `scripts/wire-tool-package.ts`. A bundle carries them only when its spec lists them. `SourcesRefresh` returns ids and counts, never text. A feature idea is a proposed work item in `tool-work`, so this package has no proposal tools of its own.
- **Bounded collection (`./repo-discovery`).** `collect()` reads the README, docs, contributing and agent-instruction files, package manifests, test file names, recent commits and the issues a person picks, each within a cap. Secret-shaped, binary and oversized files, and anything outside the repository, are left out with a reason, and the result is an inclusion and omission manifest. A person sees both lists and can exclude any source before a model sees it.
- **No execution.** Discovery installs, builds and tests nothing, and never runs repository code. Git runs only through `tool-git`'s `./run` and its hardening. It clones and fetches only through `tool-work`'s managed clone, and never pushes.
- **Classification at `source`.** Each excerpt or extracted chunk is classified at TrustOrigin `source` before it is stored, so before reflection or any tool reads it. Stored text is checked again at `source`, row by row, inside `SourceList`, `SourceGet` and `ProjectMap`, as recalled wiki text is at `memory`; those tools set `classifyOutput: false`, so the same text is not checked again at `tool`. A malicious excerpt is stored as the redaction notice and its source listed as left out; a suspicious one is kept, tagged and shown with a warning.
- **Other sources.** Uploads, including PDF, DOCX and `.ics` text through `tool-docs`' readers, registered with `tool-document-ingest`'s `registerDocumentParser` (an encrypted, oversized or unsupported file is left out with a reason); links through `tool-fetch`; examples of an input and the output wanted; and a privacy-filtered usage feed that keeps event name, count, date and an optional version and drops any other field. Without one, the record says "Usage unknown".
- **The project record.** Under `.crewhaus/run/project/`: each source with when it happened and when it was seen, owner settings, and deterministic renders. `ProjectMap` and `renderReport` are written only by deterministic code from host data.
- **Outcomes.** `OutcomesRefresh` reads `tool-work`'s one PR outcome tracker and never recomputes it.

Does not own: the managed clone, its credentials and the outcome tracker (`tool-work`, brief 333); work items and proposals (brief 333); the connection store and binding file (`control-records`, brief 324); the setup session, reflection and generation (`creation-engine`, brief 337); the GitHub App and webhook receiver (`github-app`, brief 338); the `source` origin's policy row (`boundary-classifier`); and the `repositories:` key ([#514](https://github.com/crewhaus/factory/issues/514)).

## Inputs and Outputs

Inputs: a repository through the managed clone; a person's choices of issues and exclusions; uploaded files and links; a usage feed; caps; and, in a harness, tool calls from the agent.

Outputs: a collection manifest, stored excerpts, source lists with times and folded duplicates, a codebase map, a weekly report whose claims link to their sources, and content-free ids and counts to the model from `SourcesRefresh`. Text reaches a model only through a tool that checked it, or through reflection after the collector did.

## Dependency Notes

Depends on `@crewhaus/tool-builder`, `@crewhaus/tool-catalog`, `@crewhaus/boundary-classifier`, `@crewhaus/tool-git`'s `./run`, `@crewhaus/tool-codehost`, `@crewhaus/tool-document-ingest` (an exported pure reader, with `registerDocumentParser`), `@crewhaus/tool-docs` (its PDF, DOCX and `.ics` readers, a pure subpath, under the upload caps), `@crewhaus/tool-fetch`, `@crewhaus/control-records` and `@crewhaus/tool-work` (`./workspace` for clone and fetch only, `./work-items` for outcomes). Discovery needs `git` on the host, 2.31 or later for a hosted repository. No new third-party dependency, and no spec key: a spec that lists none of its tools compiles to the same bytes.

## First Implementation Slice

`feat(tool-project): repository connections, bounded no-exec discovery and the project record` adds the package with `./repo-discovery`, the tools and the origin's first sites. Later PRs extend it, with the usage-feed source next. This brief lands in crewhaus/docs before the package does.

## Study References

`packages/tool-git/src/index.ts` (the no-remote boundary and hardened runner); `packages/tool-document-ingest/src/index.ts` (the parser seam); `packages/tool-wiki/src/index.ts` (re-checking stored text row by row); `packages/tool-fetch` (bounded fetching); and [#515](https://github.com/crewhaus/factory/issues/515).

## Validation Plan

Catalog tests: T1, T3, T8. Primary risks: **repository code running during discovery**: a fixture repository with hooks, install scripts and a build step planted proves none of them ran; **an instruction planted in project text**: a red-team suite drives a fake model that obeys instructions planted in a README, an issue, docs and a manifest, and asserts that no authority field changes and no planted instruction reaches generated instructions; **a secret collected as an excerpt**: secret-shaped files are left out and listed; **a report inventing a claim**: every claim links to a dated source.

Definition of done: `bun test packages/tool-project/src` green; a discovery run leaves the repository byte-identical; the manifest lists every file left out with its reason; `crewhaus doctor --philosophy-alignment` exits 0 with each site listed; and a spec that lists no project tool compiles to the same bytes.
