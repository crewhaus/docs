# default-skills

Status: implemented and tested
Dependency phase: 13 - Hooks/skills/commands
Catalog layer: R9 - Hooks, Skills, Slash Commands
Origin in ordering: v0.3.0 memory release (design §2.6, PR 12)
Workspace home: packages/default-skills
Targets: CLI, CHN, MGD, RES, CRW
Test layers: T1

## Purpose

The product's first shipped skills — embedded at compile time so compiled-bundle behavior never depends on the deploy machine's node_modules. Three skills: `continuity` (read the plan first, pin user requirements as verbatim REQ entries, claimed-vs-proven status honesty with `PlanComplete` toolUseId evidence, bias to action, accurate handoffs), `learning-loop` (the expert demo's ANSWER/STUDY/REFLECT/EXAM modes productized, templated at compile time via `renderSkill` with `{{domain}}`/`{{curriculum}}`/`{{sources}}` — strict both ways, so a missing or typo'd substitution throws), and `dream` (the consolidation playbook consumed by scheduled dream ticks). Plus eleven builtin slash commands: `/plan`, `/focus <text>`, `/next`, `/handoff`, `/clear-plan`, `/clear-focus`, `/forget <query>`, `/study`, `/reflect`, `/exam`, `/dream`.

## Boundaries

Owns: the skill bodies (exported both as string constants for compile-time embedding and as real SKILL.md files for runtime discovery), the command definitions, and a content-lint test pinning every tool name the bodies mention to the real v0.3.0 tool vocabulary.

Does not own: precedence/merging (`skills-registry`'s `builtinSkills` option — user `~/.crewhaus/skills` and project `.crewhaus/skills` override by name, an empty-body override disables one; `slash-commands`' `builtinDirs` + user root), classification (bodies pass the same `skill`-TrustOrigin gate as any other skill), or gating (which skills/commands wire in per spec is `memory-service`'s call).

## Inputs and Outputs

Inputs: template substitutions for `learning-loop` (domain/curriculum/sources).

Outputs: skill bodies + command definitions consumed by `memory-service`, target emitters (compile-time embedding), and the interpreter path (runtime builtin root).

## Dependency Notes

Depends on `@crewhaus/errors` only — deliberately leaf-like so emitters can embed bodies without pulling runtime machinery. The `target-claude-plugin` `renderSkill` precedent is the embedding template.

## First Implementation Slice

Shipped in factory PR 12; emitter/interpreter wiring landed with PR 10/11; `learning-loop` substitution + `/study` `/reflect` `/exam` gating landed with PR 17; the `dream` skill joins the gated set when a dream schedule is configured (PR 14).

## Study References

`skills-registry` (the dead `pluginDirs` this replaces with a real merge root), `target-claude-plugin` (`renderSkill`), the expert starter's pre-0.3.0 mechanism prompt (what these skills productize); design doc §2.6.
