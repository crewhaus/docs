# Tools Reference

> What your agent can already do, what else it could do, and how to find a
> specific tool without reading a registry.
>
> CrewHaus 0.7.0 ships **549 builtin tools**, up from 22 in 0.6.0, and a
> category grammar for turning them on a group at a time. This page covers
> finding them, enabling them, and the things they deliberately refuse to do.
>
> Everything below is a real command with real (trimmed) output. Run them
> yourself — the CLI reads the same registry the compiler does, so it is
> never out of date with your installed version.

---

## Table of contents

1. [What a builtin tool is](#what-a-builtin-tool-is)
2. [Finding a tool from the CLI](#finding-a-tool-from-the-cli)
3. [Turning tools on in a spec](#turning-tools-on-in-a-spec)
4. [What is available](#what-is-available)
5. [What these tools will not do](#what-these-tools-will-not-do)
6. [Where to go next](#where-to-go-next)

---

## What a builtin tool is

A builtin tool takes a typed input, does the work **in-process**, and returns
a result. Almost none of them makes a model call, and that is the point: the
work costs nothing and cannot drift between runs — `MoneyAllocate` splits a
bill the same way on Tuesday as it did on Monday, and `CsvParse` does not
occasionally corrupt a quoted address field the way a model reading a
spreadsheet export eventually will.

Two kinds of builtin do cost something, and `show` gives both away in the
first line of the description: `imageGenerate` sends its prompt to a remote
image model, and a few others call a paid service — `webSearch` is the
obvious one. Read the description before you assume a builtin is free.

The honest caveat: a tool's *definition* — its name, description and input
schema — is sent to the model with every request, so turning on a large
category has a context cost even when nothing calls it. Running the tool is
free; offering it is not. That is the trade the category grammar below exists
to let you make deliberately.

Builtins are distinct from tools a target shape wires in (`Retrieve` for RAG,
`Handoff` for crews) and from MCP server tools, which arrive namespaced as
`mcp__<server>__<tool>`. Only builtins appear in `crewhaus tools`.

---

## Finding a tool from the CLI

Six actions live under `crewhaus tools`. Any of them plus `--help` prints the
namespace summary — `crewhaus tools list --help` works, a bare
`crewhaus tools --help` does not, because the action is parsed first.

```
usage: crewhaus tools <list|categories|show|search|suggest|audit>

  categories               every tool category + what it turns on
  show <tool>              one tool in full: flags, categories, inputs
  search <query>           find a tool by name, description or category
  list [--category NAME]   print every builtin tool + its metadata
  suggest [spec.yaml]      rank builtins against agent.instructions
                           (deterministic keyword match; default spec
                           is ./crewhaus.yaml)
  audit [--sessions N|all] mine tool_stats across sessions vs. the
                           spec's tools: grants — unused / failing /
                           learned-readOnly (advice-only; tools: is not
                           optimizer-whitelisted)

  --json  machine-readable output
```

### `crewhaus tools search` — start here

The question people actually arrive with is "I want to do X, is there a tool
for it?" `search` answers it. It matches on name, description **and**
category, and tells you which of the three hit.

```sh
crewhaus tools search csv
```

```
10 match(es) for "csv":
  csvParse (CsvParse)  [name+description]
    Parse RFC 4180 CSV into records or rows, handling quoted fields, embedded
    commas, newlines and doubled quotes, with a custom delimiter and optional
    type inference. Use to read a spreadsheet export correctly instead of
    splitting on commas and corrupting every quoted address.
  csvWrite (CsvWrite)  [name+description]
    Render records or rows as RFC 4180 CSV, quoting any field that contains
    the delimiter, a quote or a newline. …
  exportCsv (ExportCsv)  [name+description]
    Write a query's rows to a CSV file inside the workspace, streaming them
    rather than holding them in memory. …
```

Because the category name is searched too, a capability word finds a whole
area even when no tool is named after it — `crewhaus tools search desktop`
returns nine hits, of which only `desktopNotify` matches on its name.

A search with no hits says so plainly rather than guessing:

```sh
crewhaus tools search "sign transaction"
```

```
no builtin tool matches "sign transaction" — try `crewhaus tools categories`
```

That one is not a gap in the index. Nothing in the 549 signs a transaction;
see [What these tools will not do](#what-these-tools-will-not-do).

### `crewhaus tools show` — one tool in full

Once you have a candidate name, `show` gives you the decision you actually
need to make: what it does, what it is allowed to touch, what it takes, and
the exact line to paste into a spec.

```sh
crewhaus tools show gitCommit
```

```
gitCommit  (GitCommit)
  Commit what is staged, or only the named paths, with a message and an
  optional author and date. Use it to record a change; it never amends unless
  `amend` is set explicitly, so an existing commit is never rewritten by
  accident.

  flags       mutating, destructive, external, io:process
  categories  all-code, all-git
  input       allowEmpty, amend, author, cwd, date, message, paths, timeout

  enable with  tools: [gitCommit]
```

Read the flags line before you grant anything. In `default` mode every tool
asks the first time it is called; `destructive` is what keeps a tool asking in
`auto` mode, where read-only and ordinary calls go through without a prompt,
and `plan` mode denies anything not read-only outright. `external` marks the
tool as a sink that leaves the system, so the egress classifier inspects what
it is about to transmit before the call fires; the separate `io:` flag records
the fact of what it touches — a socket (`io:network`) or a child process
(`io:process`). `justification-gated` means the model has to state why before
the call is allowed. Some tools carry more:

```sh
crewhaus tools show packageInstall
```

```
  flags       mutating, destructive, external, io:process, justification-gated
  categories  all-operations, all-pkgmgr
  input       dryRun, manager, name, timeoutMs, version
```

The two casings in the header are load-bearing. The camelCase key
(`gitCommit`) is what you write in a spec's `tools:` list; the PascalCase
name (`GitCommit`) is what session logs, traces and permission rules record.
`show` prints both so you never have to guess which one a document meant.

A typo gets a nearest-match hint rather than a dead end:

```sh
crewhaus tools show gitcommit
```

```
crewhaus: no builtin tool named "gitcommit" — did you mean gitStatus, gitDiff, gitLog?
run `crewhaus tools list` to see them all
```

### `crewhaus tools list` — everything, or one category

`list` prints every builtin with its key, runtime name, flags and
description. All 549 is a lot of scrollback, so reach for `--category` once
you know roughly where you are:

```sh
crewhaus tools list --category secrets
```

```
3 builtin tool(s) in all-secrets:
envFileUpsert (EnvFileUpsert) [destructive, external, io:process]
  Set or comment out keys in a .env file, in place, preserving every comment,
  blank line and the operator's key order. …
secretLookup (SecretLookup) [read-only, external, io:process]
  Check whether a secret reference resolves, and report where it resolves
  FROM — without returning the secret. …
secretRotate (SecretRotate) [destructive, external, io:process]
  Replace a stored secret with a new value and prove the new one reads back,
  without either value appearing in the result. …
```

The `all-` prefix is optional here: `--category secrets` and
`--category all-secrets` do the same thing.

### `crewhaus tools categories` — the map

`categories` is the one command to run if you are new. It prints every leaf
category with its tool count, the keys it owns, and a one-line summary of the
kind of work it does — then every roll-up and what it expands to.

```sh
crewhaus tools categories
```

```
categories:
  all-data  (23)  Parse, query, reshape and convert JSON, YAML, TOML, CSV and XML
    columnsToRecords, csvParse, csvWrite, dataConvert, dataDiff, dataShape, …
  all-git  (27)  Read and change a git repository: status, diffs, history,
                 blame, branches, commits, stashes and worktrees
    gitAdd, gitApplyPatch, gitBlame, gitBranchCreate, gitBranchDelete, …
  all-secrets  (3)  Resolve secret references and maintain .env files,
                    reporting presence and provenance rather than values
    envFileUpsert, secretLookup, secretRotate
  …

roll-ups:
  all-code  (143)  Everything for working in a codebase
    = all-fs + all-codegraph + all-process + all-code-exec + all-git + all-fsx
      + all-proc + all-codehost + all-toolchain + all-packaging + all-changeset
      + all-buildperf + all-registry + all-supplychain + all-containers
      + all-distribution
  all-compute  (130)  Everything a harness can do with no I/O at all — pure,
                      in-process, zero tokens
    = all-text + all-data + all-encode + all-datetime + all-schema + all-math
      + all-flow + all-onchain
  …

use in a spec:  tools: [all-fs, -write]   # a category, minus one tool
```

### `crewhaus tools suggest` — what your spec is missing

`suggest` reads a spec's `agent.instructions` and names builtins the
instructions imply but the `tools:` list does not grant. It is a deterministic
keyword match, not a model, which is what makes it safe to run in CI:

```sh
crewhaus tools suggest ./crewhaus.yaml
```

```
implied but not in tools:
  + citationLint (CitationLint) — matched: sources
  + coverageSummary (CoverageSummary) — matched: coverage
heuristic: literal keyword match over agent.instructions, not a model —
wording it doesn't recognize won't be suggested; `crewhaus tools list` shows
every builtin
```

The footer is the important part: a miss means the keywords did not fire, not
that no tool exists. Fall back to `search`.

### `crewhaus tools audit` — what the grants got wrong

`audit` goes the other way. It mines `tool_stats` and `tool_use` events out of
recent session logs and compares observed calls against what the spec granted:
tools granted and never called, tools failing chronically, and tools whose
clean call history suggests they could be treated as read-only.

```sh
crewhaus tools audit
```

```
tools audit: 1 finding(s) across 5 session(s)
[failing] webFetch (WebFetch) — 8/9 calls errored (89%); investigate inputs/backend or swap the tool
```

A well-matched spec says so:

```
tools audit: 0 finding(s) across 2 session(s)
no tool-usage findings — grants look well-matched to observed calls
```

It runs from a harness directory, reads `./.crewhaus/sessions`, and takes
`--sessions N|all` to widen the window past the default. It is **advice only**:
`tools:` is not on the optimizer's writable-paths list, so nothing here is ever
applied automatically.

### `--json` on any of them

Every action takes `--json` and prints the same data machine-readably —
`{ tools: [...] }`, `{ categories: [...] }`, `{ query, hits }`, or one tool
object. This is the seam to build on if you want a tool picker, a lint rule,
or a dashboard that shows an operator what a harness could be granted.

---

## Turning tools on in a spec

A spec's `tools:` list accepts three kinds of entry:

```yaml
tools:
  - all-git          # every tool in a category
  - webFetch         # one tool by name
  - -gitCommit       # minus one tool
```

The list lives wherever that shape keeps it — top-level `tools:` for `cli`,
`agent.tools:` for `channel`, per-step or per-role `tools:` for `workflow`
and `crew`. The grammar is expanded once at lower() time in **every**
`tools:` list a spec carries, including graph nodes, sub-agents and model
profiles, so it works identically everywhere.

### Union first, then subtract

The rule is worth learning because it makes the list safe to read:

1. Every include — `all-<category>` or a bare tool key — is **unioned**.
2. Every exclude — `-<tool>` or `-all-<category>` — is then **subtracted**.

Excludes therefore always win, no matter where they appear. You never have to
simulate the list top to bottom to know what an agent ended up with, and
moving a line cannot change the result. Writing a tool as both an include and
an exclude removes it.

```yaml
tools:
  - all-fs          # read, write, edit, glob, grep
  - all-git         # the whole git surface
  - -write          # …but no file writes
  - -gitCommit      # …and no commits
  - webFetch        # plus one extra by name
```

That spec lints clean and grants exactly what you would expect.

### Leaves and roll-ups

A **leaf** category owns tool keys directly, and every builtin belongs to
exactly one leaf — `all-git` is the git surface, `all-secrets` is the three
secret tools. Most leaves are a single `@crewhaus/tool-*` package; a few
gather more than one. A **roll-up** owns other categories and expands
transitively, so an operator can write `all-code` instead of naming every
category underneath it. `crewhaus tools categories` prints the leaves first,
then the roll-ups with their `=` expansions.

Roll-ups are where the context cost bites. `all-code` is 143 tool
definitions in every request. Prefer the narrowest category that covers the
job, and use `show` to check whether a single key would do.

### Two mistakes that fail the compile

Both of these are errors rather than warnings, on purpose.

**An unknown category.** A typo would otherwise silently grant nothing:

```yaml
tools:
  - all-filesytem
```

```
✗ [lower] <lower>: tools: unknown tool category "all-filesytem". Known
categories: all-approvals, all-buildperf, all-chain, all-chaincall, …

lint: 1 error(s), 0 warning(s).
```

**An exclusion that removes nothing.** An exclusion that matches no included
tool is almost always a typo or a stale copy-paste — the category it was
guarding got removed and the `-` line stayed. Ignoring it would leave the
author believing a tool is gated when it never was:

```yaml
tools:
  - all-fs
  - -gitCommit
```

```
✗ [lower] <lower>: tools: "-gitCommit" excludes a tool that nothing includes.
Remove the exclusion, or add the category that provides it. Currently
included: edit, glob, grep, read, write

lint: 1 error(s), 0 warning(s).
```

Catch both with `crewhaus lint <spec.yaml>` before you compile.

---

## What is available

Grouped by the kind of work rather than by package. Run
`crewhaus tools categories` for the full list with counts — this is the shape,
not the index.

**Pure computation, no I/O at all.** Text transformation and measurement,
structured data (JSON, YAML, TOML, CSV, XML), encoding, hashing and
identifiers, dates and schedules, maths and exact integer money, schema
validation, control flow, tabular intake. Nothing here opens a socket or
touches a disk. The `all-compute` roll-up collects them.

**The working tree and the toolchain.** Files, trees and archives; processes
and background jobs; git; GitHub and GitLab; SQLite; documents (Word, Excel,
PowerPoint, PDF, email, calendar); HTML without a browser; packaging,
lockfiles and registries; code intelligence; and the gates you run before a
change lands. The `all-code` roll-up collects most of it; `all-filesystem`
and `all-data-stores` are narrower cuts.

**What a harness remembers and what it reports.** Durable key-value state,
counters, checkpoints, journals, notes and lexical search; secrets handling;
messaging and notification; observability, cost, budgets, service levels and
incidents. See `all-memory`, `all-safety`, `all-outreach` and `all-obs`.

**Money and chains.** ABI encoding and EIP-712 digests offline; live EVM
block, transaction, log and gas reads; token and DeFi queries; a hash-chained
double-entry ledger; e-invoices, payment files and counterparty checks. See
`all-chain` and the money and ledger categories.

**The operator's own machine.** System and network facts, listening ports,
the host scheduler (crontab, launchd, systemd timers), filesystem watches and
the OS file index, the system package manager, and the desktop itself —
clipboard, notifications, printing, windows, presence. Every desktop tool
fails closed on a headless host with a typed reason.

**CrewHaus operating itself.** Specs and spec patches, evals and baselines,
datasets, approvals, harness lifecycle, fleets, deployments, model routing and
discovery. The `all-operations` roll-up collects them.

---

## What these tools will not do

The refusals are a feature, and they are what makes a broad grant reasonable.
Each of these is enforced in code, not documented as a convention.

**No tool among the 549 signs or sends a transaction.** The chain tools read.
Every method name goes through a shared read-only allow-list at the RPC seam
before a socket is opened, so `eth_sendTransaction` and
`eth_sendRawTransaction` throw there even if a future edit asked for them —
structural, not a promise about call sites. `crewhaus tools search "sign
transaction"` returns nothing, which is the registry agreeing.

*The honest boundary:* `@crewhaus/tool-evm-tx` **does** sign. Its
`EvmSendTransaction` runs an unsigned transaction through the wallet engine
(simulate → policy check → approval → custody sign → broadcast). It is wired
per shape, it is not in any category, and it is not one of the 549 — which is
why it does not appear in `crewhaus tools list`.

**Nothing moves money.** `PaymentFileBuild` assembles a bank-ready NACHA or
SEPA pain.001 batch with every control figure computed from the rows, and
`InvoiceRender` renders an invoice with a gap-free document number.
Transmitting either is a human's job through their own bank: neither package
carries any transport, and no schema accepts a credential, a key or a token.

**Nothing acquires privilege.** `PackageInstall` runs no `sudo`, `doas`,
`runas` or `pkexec` and raises no UAC prompt. A manager that needs root — apt,
dnf, pacman — is refused unless the process is already root, and the refusal
hands back the exact command an operator would run themselves. Homebrew, which
needs no root, proceeds. Windows installs are refused outright because both
managers end in an elevation prompt.

**`SecretLookup` does not return the secret.** A tool result reaches a model's
context, then a transcript, then a trace and a log. So it reports presence,
the backend that answered, the source, the length and a truncated SHA-256
fingerprint — enough to compare two secrets or confirm a rotation took,
without either value existing outside the tool. There is deliberately no
reveal option.

**Nothing a caller supplies becomes program text.** `osascript -e` compiles
AppleScript and a Windows toast body is an XML document inside a PowerShell
script, so a naive notification body containing a quote would *run*. In
`@crewhaus/tool-desktop` every script is a frozen module constant and caller
values travel beside the program — `item N of argv` on macOS, `$env:…` on
Windows, argv after `--` on Linux — and the escape helper refuses to build an
argv for source it did not register. There is no `sh -c` and no `cmd /c`
anywhere in the package.

---

## Where to go next

- [GETTING-STARTED.md](GETTING-STARTED.md) — the guided tour, including the
  [Tools, permissions, and skills](GETTING-STARTED.md#tools-permissions-and-skills)
  section: where the `tools:` list lives per target shape, the four permission
  modes, and how rules are layered.
- [CLI-REFERENCE.md](CLI-REFERENCE.md) — the rest of the `crewhaus` command
  surface, including `lint`, `compile` and the advise/optimize commands that
  read the same session logs `tools audit` does.
- [SKILLS-FORMAT.md](SKILLS-FORMAT.md) — skills are the other half of the
  capability story: markdown a model loads on demand, rather than a typed
  function it calls.
- [MODULE-CATALOG.md](MODULE-CATALOG.md#if-you-are-adding-a-new-tool) — if you
  need a tool that does not exist yet, the "adding a new tool" entry point has
  the packages to read and the safety floor `buildTool()` enforces.
- [crewhaus/demos](https://github.com/crewhaus/demos) — the walkthrough
  recipes, including
  [29-permissions-deep-dive](https://github.com/crewhaus/demos/blob/main/walkthroughs/29-permissions-deep-dive.md)
  for gating what you just granted.
