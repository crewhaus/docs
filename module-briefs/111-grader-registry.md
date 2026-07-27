# grader-registry

Status: implemented and tested
Dependency phase: 9 - Telemetry & eval
Catalog layer: R15 - Telemetry, Tracing, Eval
Origin in ordering: named in Part G; the `opts:` contract landed in the 0.4.x evals campaign
Workspace home: packages/grader-registry
Targets: EVAL
Test layers: T1, T5

## Purpose

Pluggable graders resolved **by name** — the indirection that lets a `graders.yaml` say `type: registry` + `grader: semantic.similarity` without the eval runner knowing anything about embedders, classifiers or OCR. The default registry the CLI falls back to installs the shipped packs and then discovers local plugins, so a project can override any pack entry without a code change.

## Boundaries

Owns: the named-grader map, the `resolveWithOpts` seam that carries a `graders.yaml` `opts:` record through to the factory, and plugin discovery. The discovery root is `<cwd>/.crewhaus/graders`, walked **last** via upsert so a plugin wins over any pack entry of the same name. There is no home-directory plugin root.

Owns the two-sided validation contract, which is deliberately asymmetric: first-party packs declare a **strict** `opts` vocabulary, and an unknown or ill-typed key is a loud error at run start naming what that pack actually accepts — while plugin graders receive the record **untouched and unvalidated** as an optional third argument, `(sample, runResult, opts?) => GradeResult`, the exported `PluginGraderWithOpts` shape. The registry never drops a plugin's opts. That escape hatch is what every "needs wiring" error message points at.

Does not own: the grader implementations themselves (`grader-nlg-metrics`, `-semantic-similarity`, `-safety-classifiers`, `-multimodal`, `grader-12-metric-rubric`, `grader-continuity`, and the runner-local `calibration.*` / `consistency.*` packs); the `graders.yaml` grammar and its strict schemas (`eval-grader`); credential resolution for judge-backed or vision-backed packs — each resolves lazily from its own env var or `opts` and throws a wiring-explaining error when neither is present.

A caller-built registry lacking the `resolveWithOpts` seam rejects opts-carrying entries loudly rather than silently dropping the options.

## Inputs and Outputs

Inputs: grader names plus an optional `opts` record; a plugin directory to discover.

Outputs: resolved grader functions, or a typed error naming the unknown grader / rejected option and the accepted vocabulary.

## Dependency Notes

Depends on `@crewhaus/eval-grader`, `@crewhaus/eval-judge` and `@crewhaus/model-adapter`. `DefaultGraderRegistry.setJudgeUsageSink(sink)` lets judge-backed registry graders meter their tokens into the run's `aggregates.judgeUsage`, so grading spend is visible next to agent spend. `run_exam` (`learning.exam.graders`) builds the same default registry `crewhaus eval` falls back to, so spec-declared exams reach the same packs, plugins and `opts:`.

## First Implementation Slice

Shipped well before 0.4.x. The evals campaign added `resolveWithOpts`, the per-pack strict opts vocabularies, the documented `PluginGraderWithOpts` third argument, the judge-usage sink, and registry-grader support inside `run_exam`.

## Study References

`helm/.../metrics/`; `lm-evaluation-harness/lm_eval/api/metrics.py`; `dspy/.../evaluate/metrics.py`; `ragas/metrics/`; OpenAI graders.

Research focus: Composition; calibration; the plugin / first-party validation boundary.

## Validation Plan

Catalog tests: T1, T5. Primary risks: quality regressions that only show up through eval or trace grading, and silently-dropped configuration — an `opts` key that looks honoured and is not, the failure the strict vocabularies exist to make loud.

Definition of done: tests are green, public types are exported from the intended workspace, failure modes use typed `CrewhausError`-style errors where applicable, and the catalog status can be updated without hand-waving.
