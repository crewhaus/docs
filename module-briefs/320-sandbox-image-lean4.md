# sandbox-image-lean4

Status: implemented and tested
Dependency phase: 7 - Permission/policy
Catalog layer: R8 - Permission, Policy, Safety
Origin in ordering: 0.8.0 — the provability release (design §6.4, §13, §17 decision 4)
Workspace home: packages/sandbox-image-lean4 (the registration, and the `Dockerfile` the image is built from)
Targets: CLI, CHN, MGD (any harness whose `provability.formal.checkers` names `lean4` and that has a docker or podman sandbox)
Test layers: T1, T2, T7, T8

## Purpose

The curated Lean 4 image behind the provability ladder's `lean4` checker (`proof-checker`), registered into `sandbox-image-registry` exactly the way `sandbox-image-go` registers Go.

It is **Mathlib-free**. A Mathlib image is multi-gigabyte, and most theorem-shaped claims a harness makes are finite or arithmetic, which the `builtin` and `smt` checkers already cover. The curated image carries the Lean toolchain alone, so proofs may import `Init` and `Std`. A spec that needs Mathlib points `provability.formal.image` at its own image, which must be on the sandbox's allowlist, and the lean4 adapter then also accepts `import Mathlib…`. The design first planned no curated image at all; one is curated because no suitable upstream image exists.

## Boundaries

Owns:

- **The pinned reference** — `LEAN4_IMAGE_REF` (`ghcr.io/crewhaus/sandbox-lean4` at `LEAN4_IMAGE_TAG`), the Lean release the image carries (`LEAN4_TOOLCHAIN_VERSION`, held in step with the Dockerfile by a test), the default entrypoint `lean --stdin --json` (one file on stdin, messages back as JSON lines), the healthcheck `lean --version`, and the looser cold-start budget a large toolchain needs. The tag is a constant, not the package version read at runtime, for the same reason as Z3's: a compiled `crewhaus` binary cannot read its package.json, and a pinned tag keeps every release on a known toolchain.
- **The Dockerfile** — the official Lean 4 release for the build's architecture, checked against a sha256 pinned per architecture, cut down to what `lean --stdin --json` needs to elaborate a file importing `Init` and `Std`: `lean`, the shared libraries it links, and the Init and Std modules' compiled files, plus `cadical`, the SAT solver Std's `bv_decide` runs. Left out: the bundled clang and linker, static libraries, lake, the Lean and Lake modules and editor indexes — most of the toolchain. The fetch stage elaborates one file importing Std with the pruned tree (a `bv_decide` goal among it), so a toolchain bump that needs another file fails the build rather than the first proof. The shipped stage is distroless `cc-debian13` — no shell, no package manager, no network tools — running as uid 65532, with bases pinned by digest and an empty build context. Not elan: elan resolves toolchains over the network at first use and wants a writable home.
- **Registration** — `registerLean4SandboxImage()` (throws on a second call) and the idempotent `ensureLean4SandboxImage()`, which put the ref on `listAllowedImageRefs()` for a caller that builds its sandbox from the registry's allowlist.

Does not own: the file Lean elaborates — `proof-checker`'s lean4 adapter writes it, with the model's statement and proof inside one `theorem` after a deterministic pre-scan, followed by the adapter's own `#print axioms`; the axiom policy (the adapter's: a `bv_decide` proof trusts the compiler, so it is accepted only when the harness drops `native_decide` from `formal.forbid`); the container runtime and its defaults (`sandbox`); and building and publishing the image (the release workflow's `sandbox-images` job, and `scripts/build-sandbox-images.ts lean4` locally).

## Inputs and Outputs

Inputs: none at run time beyond the registration call.

Outputs: a `SandboxImageEntry` in the registry, and the published image.

## Dependency Notes

Depends on `@crewhaus/sandbox-image-registry`, `@crewhaus/sandbox` and `@crewhaus/errors`. `proof-checker` imports the pinned ref (re-exported as `DEFAULT_LEAN4_IMAGE`), and `tool-proof` puts it — or the spec's `formal.image` override — on the allowlist of the sandbox it creates for the checkers.

`lean --stdin` reads its file from stdin and writes no compiled output, so the sandbox's read-only root costs it nothing. The release's `sandbox-images` job builds and smoke-runs the image the way the sandbox runs it before pushing, and never overwrites a published version tag, as it does for Z3. It publishes the release runner's architecture (amd64), so other hosts run the image emulated.

## First Implementation Slice

The registration constants came first, in the release's first stage beside the lean4 adapter and its pre-scan, with the adapter's output parsing tested against scripted Lean output; the Dockerfile, the local build script and the release job came with the third stage, and the adapter was then held to what the real toolchain prints.

## Study References

`sandbox-image-go` (the registration mould), the Lean 4 release assets and their published digests, Lean's `--json` message output and `#print axioms`, and design §6.4 and §16 (why Lean 4 is not the only checker).

## Validation Plan

Catalog tests: T1, T2, T7, T8 — the `sandbox-image-go` suite: registration shape, the allowlist round trip through the noop sandbox, the cold-start budget and escape-attempt registrations. `dockerfile.test.ts` holds the Dockerfile to its contract without Docker: the pinned version, the per-architecture sha256s, only Init and Std unpacked, bases pinned by digest, a shipped stage with no `RUN`, no shell and no root, and the registry's healthcheck argv. `CREWHAUS_SECTION36_LIVE_DOCKER=1` also runs the healthcheck against the real image, and `proof-checker`'s live suite runs proofs on it — `sorry` and the `sorryAx` backstop, `Classical.choice`, compiler-trust axioms (`bv_decide` running the image's cadical among them), and a `formal.image` override on and off the allowlist.

Primary risk: a **toolchain bump that changes what Lean prints** — its JSON messages or its axiom report — which the adapter's parsers depend on. Captured output from the curated toolchain pins the parsers in `proof-checker`'s unit tests; the build's own elaboration step and the live suite catch the rest.

Definition of done: the suite green, the image built by `bun scripts/build-sandbox-images.ts lean4` passing its healthcheck, and the lean4 checker's live proofs passing on it.
