# sandbox-image-z3

Status: implemented and tested
Dependency phase: 7 - Permission/policy
Catalog layer: R8 - Permission, Policy, Safety
Origin in ordering: 0.8.0 — the provability release (design §6.4, §13)
Workspace home: packages/sandbox-image-z3 (the registration, and the `Dockerfile` the image is built from)
Targets: CLI, CHN, MGD (any harness whose `provability.formal.checkers` names `smt` and that has a docker or podman sandbox)
Test layers: T1, T2, T7, T8

## Purpose

The curated Z3 image behind the provability ladder's `smt` checker (`proof-checker`), registered into `sandbox-image-registry` exactly the way `sandbox-image-go` registers Go.

The image is CrewHaus's own. No suitable upstream image exists, and a distro package is the wrong unit: Debian's Z3 is several releases behind, and Alpine's pinned revisions vanish from the mirror when it updates. The official release binary is a version and a hash, so the release builds `ghcr.io/crewhaus/sandbox-z3` from this package's Dockerfile and pushes it to GHCR.

## Boundaries

Owns:

- **The pinned reference** — `Z3_IMAGE_REF` (`ghcr.io/crewhaus/sandbox-z3` at `Z3_IMAGE_TAG`), the solver release the image carries (`Z3_SOLVER_VERSION`, held in step with the Dockerfile by a test), the default entrypoint `z3 -in` (one SMT-LIB 2 script on stdin), the healthcheck `z3 --version`, and a cold-start budget for one native binary. The tag is a constant, not the package version read at runtime: a compiled `crewhaus` binary cannot read its package.json, and a pinned tag keeps every release on a solver whose version is known. Bump it when the image changes.
- **The Dockerfile** — two stages. The fetch stage downloads the official Z3 release for the build's architecture (BuildKit's `TARGETARCH`; the legacy builder, which leaves it empty, is refused rather than guessed at), checks it against a sha256 pinned per architecture, and keeps the one `z3` binary and its licence. The shipped stage is distroless `cc-debian13` — glibc and libstdc++, no shell, no package manager, no network tools — running as uid 65532, with both bases pinned by digest and an empty build context (`.dockerignore`).
- **Registration** — `registerZ3SandboxImage()`, which throws on a second call through the registry's duplicate-id refusal, and the idempotent `ensureZ3SandboxImage()`, which a boot path that may run twice uses. Either puts the ref on `listAllowedImageRefs()` for a caller that builds its sandbox from the registry's allowlist.

Does not own: the script Z3 runs — `proof-checker`'s smt adapter writes it, and the model's problem goes in as data between the adapter's own `check-sat` commands, never as a program; the container runtime and its run-time defaults (`sandbox`: no network, read-only root, a `/tmp` tmpfs, memory and CPU caps, no new privileges); and building and publishing the image — the release workflow's `sandbox-images` job does that, and `scripts/build-sandbox-images.ts z3` builds the same ref locally.

A spec cannot swap this image: `provability.formal.image` overrides the lean4 image only. A library caller can, through `proof-checker`'s `createDefaultCheckers({ images: { smt } })`; that image must then be on the sandbox's allowlist too, and its Z3 has not met the smt checker's reserved-name probe.

## Inputs and Outputs

Inputs: none at run time beyond the registration call.

Outputs: a `SandboxImageEntry` in the registry, and the published image.

## Dependency Notes

Depends on `@crewhaus/sandbox-image-registry`, `@crewhaus/sandbox` and `@crewhaus/errors`. `proof-checker` imports the pinned ref (re-exported as `DEFAULT_SMT_IMAGE`), and `tool-proof` puts it on the allowlist of the sandbox it creates for the checkers, so a harness needs no registration call to run the `smt` checker.

The release's `sandbox-images` job builds the image into the runner's daemon, runs it the way `@crewhaus/sandbox` runs it (no network, read-only root, a `/tmp` tmpfs, uid 65532) on the checker's own argv and stdin, and pushes it only when that run succeeds — the Dockerfile's own checks run in the fetch stage, not in the distroless stage that ships. A published version tag is never overwritten: each release's smt adapter was proven against the solver in that exact image, so re-running the job on a newer tree must not swap in one it never met.

## First Implementation Slice

The registration constants came first, in the release's first stage beside the smt adapter, so the checker could be built and tested against a host Z3; the Dockerfile, the local build script and the release job came with the third stage, once the adapter's script contract was settled.

## Study References

`sandbox-image-go` (the registration mould), `sandbox-image-registry`'s healthcheck contract, the Z3 release assets and their published digests, and design §6.4.

## Validation Plan

Catalog tests: T1, T2, T7, T8 — the `sandbox-image-go` suite: registration shape (T1), the allowlist round trip through the noop sandbox (T2), the cold-start budget (T7) and escape-attempt registrations (T8). `dockerfile.test.ts` holds the Dockerfile to its contract without Docker: the pinned version, the per-architecture sha256s, bases pinned by digest, a shipped stage with no `RUN`, no shell and no root, and the registry's healthcheck argv. `CREWHAUS_SECTION36_LIVE_DOCKER=1` also runs the healthcheck against the real image, and `proof-checker`'s live suite runs proofs on it.

Primary risk: a **solver bump that resolves a new built-in name** the smt checker does not reserve, which a preamble could then redefine. The live suite re-probes the image's Z3 for exactly that, with known built-ins as controls, so the bump fails until the name is reserved.

Definition of done: the suite green, the image built by `bun scripts/build-sandbox-images.ts z3` passing its healthcheck, and the smt checker's live proofs passing on it.
