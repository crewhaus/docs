# qr-code

Status: implemented and tested
Dependency phase: 18 - Deployment & studio
Catalog layer: F3 - Deployment & Operations
Origin in ordering: 0.6.x — the `crewhaus hangar --lan` phone door
Workspace home: packages/qr-code
Targets: All (a CLI-side utility; it is not embedded in a compiled harness)
Test layers: T1

## Purpose

A QR Code encoder and terminal renderer with no dependencies, written for one
line of output: `crewhaus hangar --lan` prints a symbol, an operator scans it
with their phone, and the Hangar console opens.

It exists as a package rather than an install because `apps/cli` carries
exactly one non-workspace dependency by policy, and because a compiled
binary raises the stakes: `bun build --compile` embeds only what the import
graph references statically, so a QR library that loads a data file at
runtime would pass `bun test` and brick the released binary — the v0.3.2
`default-skills` lesson. This package has no data files at all. Every
lookup table is either transcribed as a literal or computed from the
symbol's own geometry.

## Boundaries

Owns:

- **`tables.ts`** — the two tables ISO/IEC 18004 does not let you derive
  (error-correction codewords per block, and block count, each indexed
  `[level][version]`), plus the capacity arithmetic that *is* derivable.
  `rawDataModules` is the spec's function-pattern accounting in closed form,
  so total and data codewords are computed rather than transcribed: three of
  the usual four columns become arithmetic, and every column not typed out is
  a column that cannot be mistyped. `alignmentCentres` carries the standard's
  one exception — version 32 spaces its patterns 26 modules apart where the
  formula says 28.
- **`galois.ts`** — GF(256) with the primitive polynomial 0x11D: log/antilog
  tables, generator polynomials, and the Reed–Solomon remainder. Nothing
  QR-specific beyond the choice of polynomial.
- **`encode.ts`** — the pipeline: version selection, the bit stream (mode,
  character count, terminator, pad codewords), block splitting and
  interleaving, function-pattern placement, the zig-zag data walk, mask
  scoring, and the BCH-protected format and version information. **Byte mode
  only**, deliberately: numeric and alphanumeric modes only pay for
  themselves alongside a segmentation optimiser, and the payloads this exists
  for (`http://host:port/#t=<hex>`) contain lowercase letters, which the
  alphanumeric character set cannot hold. The four mask-penalty rules are
  exported individually because a rule that can only be tested through their
  sum is a rule that is not really tested.
- **`render.ts`** — terminal output, which is two problems rather than one.
  A terminal cell is about twice as tall as it is wide, so the half-block
  characters `▀` and `▄` carry two module rows per text row and the modules
  come out square. And a terminal has no fixed background, so each line is
  wrapped in an explicit black-on-white SGR pair — from the 256-colour cube,
  not the 8-colour palette, whose first sixteen entries are exactly the ones
  a theme remaps. With colour off the mapping must **invert**, because the
  only remaining way to make light modules light is to draw them with the
  block characters and let the terminal's own dark background be the ink.

Does not own: where the symbol is printed or what it encodes
(`apps/cli/src/hangar-cmd.ts`), or which address goes into the URL
(`apps/cli/src/lan-address.ts`).

## Inputs and Outputs

Inputs: a string, an optional minimum EC level (default `M`), a version
range, and render options (quiet zone, colour, inversion, ASCII fallback).

Outputs: a `QrCode` (`version`, `ecLevel`, `mask`, `size`, `modules[y][x]`)
and an array of terminal lines.

The requested EC level is a **floor, not a target**: once a version is
chosen, the level is raised as far as that version allows at no cost in
symbol size, because a stronger level across the same number of modules is
free damage tolerance — which is what a symbol photographed off a screen at
an angle actually needs. `boostEcLevel: false` pins it.

## Dependency Notes

None. That is the point.

## First Implementation Slice

The whole package, alongside `crewhaus hangar --lan` and `crewhaus hangar
qr`.

## Study References

ISO/IEC 18004 for the capacity, alignment and BCH tables; the half-block
rendering convention shared by terminal QR renderers generally.

## Validation Plan

Catalog test layer: T1.

Primary risk is a **silently wrong symbol** — a mistyped table row or a
misplaced module produces a QR code that looks entirely plausible and scans
on nothing. Three things answer it.

A round-trip reader in the test suite walks the finished symbol in the
spec's zig-zag, un-masks, un-interleaves and pulls the payload back out; it
shares the capacity tables with the encoder and nothing else, so a placement,
masking or interleaving error fails a test rather than shipping.

The tables and the mask rules were cross-checked against an **independent
encoder and decoder** — macOS `CIQRCodeGenerator` and Vision's
`VNDetectBarcodesRequest` — which caught two real transcription errors that
every structural test had passed: a wrong block count at version 8 level H,
and the version 32 alignment-step exception. All 160 version × EC-level
capacity boundaries, all 799 alignment patterns from version 2 to 40, 27
symbols' format information across all four levels, and 18 symbols' version
information were confirmed against it, and rendered terminal output was
rasterised and decoded in both polarities. That oracle needs macOS, so it
cannot run in CI; its conclusions are frozen as literals in `encode.test.ts`,
and the round-trip reader re-derives the rest on every run.

Definition of done: `bun test packages/qr-code/src` green, and
`crewhaus hangar --lan` prints a symbol a phone opens.
