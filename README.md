# CrewHaus Documentation

The standalone documentation repository for [CrewHaus Factory](https://github.com/crewhaus/factory) — architecture, module catalog, module briefs, and the build roadmap.

## Contents

- [Whitepaper — *CrewHaus: A Meta-Harness Compiler for AI Agents*](whitepaper/crewhaus-meta-harness-compiler.pdf) — the positioning + architecture paper (PDF; [Markdown source](whitepaper/crewhaus-meta-harness-compiler.md))
- [GETTING-STARTED.md](GETTING-STARTED.md) — guided tour for new users, from first principles to a runnable agent
- [CLI-REFERENCE.md](CLI-REFERENCE.md) — the complete `crewhaus` command surface, grouped by task (build/run, the eval flywheel, the observer/advisor, model & cost automation, self-healing ops, deploy/govern, fleet, safety, compliance)
- [HANGAR.md](HANGAR.md) — the harness manager: the `crewhaus hangar` console, the Advisor (alerts, suggestions, reports, and the issue inbox — per harness and fleet-wide), library curation, the machine-wide registry, `crewhaus daemon`, `crewhaus.control.v1`, and the security model
- [PROVIDERS.md](PROVIDERS.md) — the canonical model-provider reference: the model-string grammar, per-provider setup, capabilities, and troubleshooting
- [WEB-UI.md](WEB-UI.md) — [`@crewhaus/ui`](https://github.com/crewhaus/ui): drop-in, shape-aware web UIs for a compiled harness — quick start, CLI, programmatic API, and how it streams `TraceEvent`s
- [chvm](https://github.com/StudioMaxIO/chvm) — the CrewHaus version manager, and **the recommended way to install `crewhaus`**: pinned side-by-side installs, instant shim-based switching (`chvm use`), plus `system` and factory-checkout (`local`) targets. Needs Bun, on macOS or Linux; Homebrew, Scoop, winget, apt, and npm remain supported alternatives
- [SKILLS-FORMAT.md](SKILLS-FORMAT.md) — the `SKILL.md` file format, discovery roots, and precedence
- [COMPILER-ARCHITECTURE.md](COMPILER-ARCHITECTURE.md) — the meta-harness compiler reference
- [AI-Harness-Systems.md](AI-Harness-Systems.md) — a survey of production agent harnesses, and the meta-harness argument the factory is built on
- [MODULE-CATALOG.md](MODULE-CATALOG.md) — full module catalog across the 25 catalog layers
- [MODULE-CATALOG-STATUS.md](MODULE-CATALOG-STATUS.md) — implementation status of catalog modules
- [module-briefs/](module-briefs/) — one brief per module (300+ files): responsibilities, dependencies, and catalog layer
- [build-roadmap.md](build-roadmap.md) — the long-form build roadmap the catalog and the briefs were derived from (a frozen narrative snapshot; its counts are point-in-time)
