# CrewHaus Skill Format

The canonical SKILL.md schema CrewHaus's [`packages/skills-registry`](../factory/packages/skills-registry) parses. The format is forward-compatible with Anthropic's vertical-pack conventions (used in [academic-research-skills](https://github.com/anthropics/academic-research-skills), [claude-for-legal](https://github.com/anthropics/claude-for-legal), [financial-services](https://github.com/anthropics/financial-services), and the broader skills ecosystem) so packs from those repos drop into a CrewHaus skills directory without translation.

This document is the source of truth for skill authors. The runtime parser, validator, and Studio's skill explorer all consume this schema.

## File layout

A skill lives in its own directory under one of three discovery roots:

```
~/.crewhaus/skills/<name>/SKILL.md                    # user-level
<cwd>/.crewhaus/skills/<name>/SKILL.md                # project-level
<plugin>/SKILL.md (via DiscoverSkillsOptions)         # plugin-bundled
```

Later entries override earlier entries by `name`. The body of each file stays on disk until the model calls `Skill({ name })` to load it — frontmatter-only at discovery is the cheap path.

Optional companion files:

```
<skill-dir>/SKILL.md                  # required — the skill body
<skill-dir>/agents/<subagent>.md      # optional — subagent system prompts
<skill-dir>/references/<schema>.json  # optional — JSON Schema envelopes
<skill-dir>/templates/<file>.json     # optional — example payloads
```

`agents/` is detected at discovery time; the registry exposes the list via `SkillRef.subAgentFiles` (v0.3+). `references/*.schema.json` is consumed by tools like `tool-validate` when the skill performs a cross-skill handoff.

## Frontmatter

The first lines of `SKILL.md` are a YAML block delimited by `---`:

```markdown
---
name: review-and-redline
description: Review a contract against a playbook and suggest playbook-relative redlines
argument-hint: "[matter] [--draft | --redline | --review]"
triggers: [redline, review, contract review]
tools: [Read, Edit, Grep, Glob]
metadata:
  version: "2.1.0"
  status: active
  task_type: closed
  related_skills: [draft-contract, validate-clause]
---

# Review and Redline

[skill body in markdown]
```

### Required fields

| Field         | Type   | Constraint     | Notes |
|---------------|--------|----------------|-------|
| `name`        | string | min length 1   | Slash-command-style canonical name. Used as the dispatch key. |
| `description` | string | min length 1   | One sentence. Surfaced in the system prompt's "Available skills" block. |

### Optional fields (CrewHaus originals)

| Field      | Type            | Notes |
|------------|-----------------|-------|
| `triggers` | array of string | Keyword matchers. The runtime may surface the skill in the system prompt only when a user message matches. Pure-keyword for v1; intent-detection in v0.3+. |
| `tools`    | array of string | Tool-name allow-list. v1 parses but does not enforce; v0.3+ adds runtime narrowing of the tool catalog while the skill is "active". |

### Optional fields (Anthropic vertical-pack interop)

| Field           | Type            | Anthropic source | Notes |
|-----------------|-----------------|------------------|-------|
| `argument-hint` | string          | `claude-for-legal/skills/*/SKILL.md` | Slash-command-style argument summary. Either `argument-hint` (kebab) or `argumentHint` (camel) accepted. Surfaced verbatim. |
| `metadata`      | object          | `academic-research-skills/skills/*/SKILL.md` | See below. |

#### `metadata` shape

```yaml
metadata:
  version: "3.7.3"          # any string; semver recommended
  status: active            # one of: draft, active, deprecated
  task_type: open-ended     # one of: open-ended, closed, hybrid
  related_skills: [other]   # cross-skill linkage for plugin marketplace + Studio
```

Both `task_type` (snake) and `taskType` (camel) are accepted; both `related_skills` and `relatedSkills` work. The registry normalizes to the camelCase shape internally.

## Composition patterns

The vertical packs document three composition idioms that CrewHaus supports:

### 1. Skill → subagent fan-out

Skills can include an `agents/` subdirectory whose files are subagent system prompts. The skill body composes them via the `Task` tool:

```markdown
# Material Passport Research Pipeline

Step 1: Hand the user query to `agents/research_architect_agent.md`
        for plan construction.
Step 2: For each research thread, spawn `agents/synthesis_agent.md`
        with the planned context.
Step 3: Validate citations via `agents/literature_verification_agent.md`.
```

The `Task` tool's sub-agent spawner uses `sub-agent-permission-inheritance` to scope the child's permissions.

### 2. Schema-typed handoffs

When two skills exchange structured data, define the envelope as a JSON Schema in `references/`:

```
review-and-redline/
├── SKILL.md
├── references/
│   └── redline.schema.json   # output envelope
```

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "Redline",
  "type": "object",
  "required": ["clause_id", "suggested_text", "rationale"],
  "properties": {
    "clause_id": { "type": "string" },
    "suggested_text": { "type": "string" },
    "rationale": { "type": "string" }
  }
}
```

The skill body references the schema; `tool-validate` enforces it on outputs.

### 3. Practice profile / CLAUDE.md cold-start

Vertical packs like `claude-for-legal` ship with a top-level `CLAUDE.md` that documents the user's practice profile (matter list, jurisdiction, citation style). On first run, an interview-style skill writes this file; subsequent skill invocations read it to configure their behavior.

CrewHaus's `runtime-core` already loads `CLAUDE.md` from the project root into the system prompt. Vertical-pack skills that depend on this convention work as-is.

## Confidence discipline

A convention from Anthropic's vertical packs that authors are encouraged to adopt: mark uncertain claims explicitly.

```markdown
The change in Section 3.2 affects [VERIFY] approximately 47 contracts.
The amount limit is [UNCERTAIN] either $50k or $100k — confirm against
the customer's signed master agreement.
```

Tags `[VERIFY]`, `[UNCERTAIN]`, and `[LOW-CONF]` are stripped from final output by some downstream consumers but preserved in audit logs. Authors should use them whenever the skill's answer hinges on a fact the model isn't certain about.

## Worked example: minimal skill

```markdown
---
name: summarize-pull-request
description: Summarize a GitHub pull request for the team standup
argumentHint: "<repo>#<number>"
triggers: [summarize PR, PR summary]
tools: [Read, WebFetch]
metadata:
  version: "1.0.0"
  status: active
  taskType: closed
---

# Summarize Pull Request

Given a `<repo>#<number>` argument:

1. Fetch the PR metadata via `WebFetch(https://api.github.com/repos/<repo>/pulls/<number>)`.
2. Identify the files touched and the summary of each commit.
3. Produce three bullets:
   - **What changed**: one sentence per file.
   - **Why**: paraphrase the PR description.
   - **Risk**: any [VERIFY] item that needs reviewer attention.

Do NOT inspect the diff line-by-line — that's the reviewer's job. Stay
under 100 words.
```

## Worked example: Anthropic vertical-pack import

A skill copied from `anthropic/claude-for-legal/skills/commercial-legal/review-and-redline/SKILL.md` parses unchanged. The kebab-cased keys (`argument-hint`, `task_type`, `related_skills`) are accepted; the registry normalizes them to the camel-cased CrewHaus canonical shape.

## Validating a skill

The registry parses on discovery. To validate without running:

```bash
crewhaus skills validate ./.crewhaus/skills/my-skill/SKILL.md
```

Outputs the parsed frontmatter as JSON or an explanatory error. Useful in CI before merging skill packs.

## Migration notes

- **v0.2.x**: schema gains `argument-hint`, `argumentHint`, `metadata` (with snake + camel key acceptance). Old skills using only `name`, `description`, `triggers`, `tools` parse unchanged.
- **v0.3+ (planned)**: `agents/` and `references/*.schema.json` subdirectory discovery surfaces on `SkillRef`. `tools` allow-list enforced at runtime when the skill is active.
- **v0.4+ (proposed)**: `metadata.status: deprecated` skills warn at discovery; Studio shows a strikethrough in the skill explorer.

## Source pointers

- Schema: [packages/skills-registry/src/index.ts](../factory/packages/skills-registry/src/index.ts)
- Tests: [packages/skills-registry/src/index.test.ts](../factory/packages/skills-registry/src/index.test.ts)
- Recipe: [demos/recipes/15-skills.md](https://github.com/crewhaus/demos/blob/main/recipes/15-skills.md)
- Anthropic vertical pack references:
  - [academic-research-skills](https://github.com/anthropics/academic-research-skills) — metadata + agents subdirectory convention
  - [claude-for-legal](https://github.com/anthropics/claude-for-legal) — argument-hint + practice-profile CLAUDE.md
  - [financial-services](https://github.com/anthropics/financial-services) — version-bump CI + dual-path skill bundling
