# claim-ledger

Status: implemented and tested
Dependency phase: 1 - Foundations
Catalog layer: R17 - Infrastructure & Cross-Cutting
Origin in ordering: 0.8.0 — the provability release (design §4.1); the substrate the gate is built on
Workspace home: packages/claim-ledger
Targets: All (a dependency-free text library; the provability gate, `tool-verify`, the continuity and faithfulness graders and the research report gate read it — it is not wired per shape)
Test layers: T1

## Purpose

The one text layer every claim checker reads: what a document claims, what each claim cites, what kind of evidence a citation names, whether a source says the claim's words, and which values a claim states. It has no dependencies and makes no model call; every answer is a function of the characters.

It exists because the same heuristics used to live in three places. `tool-verify` carried the citation grammar, the sentence splitter and the FactCrossCheck kernel; `grader-continuity` carried the content tokens and coverage; `grader-12-metric-rubric` carried a byte-for-byte copy of those because it could not declare the dependency. Two copies of a marker pattern drift, and then one tool checks a claim the other never saw. The provability gate had to see exactly the claims and markers those tools see, so the text layer got one owner, and the three import it now.

## Boundaries

Owns:

- **The citation grammar** (`citations.ts`) — markers (`[1]`, `[^a]`, `[A1]`) and definitions (`[1]: source`, with the source also allowed on the next line): `lintCitations`, `citationDefinitions`, `citedClaims`. Both patterns are module-private. The marker regex (`CITATION_MARKER`) has one definition in the tree, and `crewhaus doctor --philosophy-alignment` reports a second one in any package's or app's source or in `scripts/` (test files aside). The definition rule (`DEFINITION`) is read through `citationDefinitions` and has no doctor check.
- **Claim extraction** (`extractClaims`) — every declarative, non-question sentence that carries a citation marker, states a number, or has at least `MIN_CLAIM_TOKENS` content words. Code, title headings and definition lines are not claims; a heading is one when it cites a source for a figure or a claim's words, states a measured value after a label (`## Invoice total: 1,284.50`), or is a sentence stating a figure (`## Refunds close after 30 days`). The extractor reads code and lists as a CommonMark renderer does, gives a marker written after the full stop to the sentence before it, and cuts a claim from a question that trails it in the same sentence (`…30 days [1] — anything else?`). Every place it differs from the older `citedClaims` leans toward reading a sentence, because one the extractor skips is one the gate passes unread. `citedClaims` keeps its own extraction, so the claims FactCrossCheck reads from a document do not move.
- **The evidence-reference grammar** (`refs.ts`) — `classifyDefinition` says what a definition's source names: a `url`, a `doi`, a workspace `path` (with its `#fragment`), `tool:<toolUseId>`, `proof:<prf_id>`, `assume:<text>`, or free `text`. It is backward compatible by construction: the definition pattern did not change, so every existing document parses as before, and the new prefixes are simply sources the older tools could not locate. `definitionGloss` returns what a definition line shows past what it names, which the gate reads as claims citing that definition.
- **The locate kernel** (`locate.ts`, FactCrossCheck's, moved here) — `indexSource` a body once, then `locateClaim` returns `supported`, `misquoted`, `polarityConflict` or `notFound`, with the excerpt, line, coverage and a `cause`. `coverage` and `windowTokens` are parameters: FactCrossCheck locates at `coverage: 1`, as it always has; the gate passes its `grounded.coverage`. Below 1 the figure rule holds — some of a claim's words may be missing, never a figure and never a negation the claim makes — and a quotation of four or more words is checked verbatim whatever the coverage. `isWaivableMiss` says whether a miss is a pure paraphrase miss (`cause: "coverage"`), the only kind the gate's waiver judge may be asked about.
- **Numbers** (`numbers.ts`) — `extractNumbers` and `containsValue` compare values as exact decimals, so `1,284.50`, `1284.50` and `1284.5` are one value, and an ISO date and a time of day count as whole values too. What cannot be read with certainty (a version, an IP address, a locale-dependent separator) is left out rather than read as the wrong value.
- **The fold** (`fold.ts`) — one character-for-character fold that every comparison reads through: curly quotes, a digit of any script, fullwidth and mathematical letters and digits, a Cyrillic or Greek lookalike inside a Latin word, and an ignorable character hidden in a word. It keeps every length, so the offsets the kernel reports are the text's own, and an excerpt is cut from the text as written.
- **The graders' measure** — `contentTokens`, `coverage`, `sentences`, `claimSentences` and `messageText`. The continuity and faithfulness graders' thresholds were set against exactly these. The locate kernel compares with a second measure, `claimWords` (Unicode words, numbers with their separators, negations kept), because "is this about the same thing?" and "does the source say these words?" are different questions.

Does not own: what each kind of evidence needs before it counts at a rung (`provability`); opening, fetching or resolving anything — nothing here reads a file, a URL or a log, and the kind of a source is read off its characters; the evidence index, the proof bundle, or any verdict about a whole answer.

## Inputs and Outputs

Inputs: text — an answer, a definition line, a source body — and options (`coverage`, `windowTokens`, `minTokens`, `bibliographic`).

Outputs: extracted claims with their markers and lines, citation reports and definition maps, classified evidence references, locations (status, cause, excerpt, coverage), and normalized numbers and quotations.

## Dependency Notes

None, by design: the dependency-free discipline of `tool-categories` and `qr-code`.

Consumers: `provability` (every text operation the verifiers perform is this package's), `tool-verify` (CitationLint and FactCrossCheck; its `lib/markdown.ts` and `lib/spans.ts` now re-export from here, so its public API did not change), `grader-continuity`, `grader-12-metric-rubric`, `report-writer` and `apps/cli`.

The move is behaviour-neutral for ASCII text. The one deliberate change is the fold: FactCrossCheck now compares a claim and its source through it, in every harness that grants the tool, so a claim 0.7.x reported `notFound` against a source spelt with lookalike characters can now be `supported`, and one it reported `supported` against a `not` hiding a zero-width character is now a `polarityConflict`. The CHANGELOG lists it among the changes that affect a harness without `provability:`.

## First Implementation Slice

Built in the release's first stage, ahead of everything that reads it: the grammar, the splitter, the locate kernel and the token measures moved out of `tool-verify` and the two grader packages, which import them back, and `extractClaims`, the evidence-reference grammar and the number comparison were new beside them. The extractor's handling of headings, trailing questions, definition glosses and lists, and the fold, grew as the release's reviews found shapes the gate read wrongly; each one is pinned in `claims.test.ts`, `locate.test.ts` or `fold.test.ts`.

## Study References

`tool-verify`'s CitationLint and FactCrossCheck (where the grammar and the kernel came from), `grader-continuity`'s heuristics and `grader-12-metric-rubric`'s faithfulness graders (the token measures), the CommonMark spec (link reference definitions, fences, list items), and design §2 (what counts as a claim) and §4.1.

## Validation Plan

Catalog tests: T1. Primary risks: **a claim the extractor does not see**, which the gate then passes unread, and **two consumers disagreeing** about what a document claims or cites. The first is answered case by case in the extractor's tests and, independently, by provability's vacuous-pass backstop, which does not rest on the extractor alone; the second by the one marker regex and the doctor finding that enforces it.

Definition of done: `bun test packages/claim-ledger/src` green, the consumers' suites green, no dependencies, and `doctor --philosophy-alignment` green on the citation-marker finding.
