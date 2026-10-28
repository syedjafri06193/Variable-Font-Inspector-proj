# Variable Font Inspector — Design & Build Guide

**Project:** Specimen tool for auditing variable axes, kerning pairs, and OpenType feature coverage across arbitrary sample text and weights
**Language:** TypeScript
**Status of this document:** planning + reference

---

## Table of contents

1. [Executive summary and scope](#1-executive-summary-and-scope)
2. [Reality check](#2-reality-check)
3. [The shaping foundation](#3-the-shaping-foundation)
4. [Variable font mechanics](#4-variable-font-mechanics)
5. [The kerning audit](#5-the-kerning-audit)
6. [The feature coverage audit](#6-the-feature-coverage-audit)
7. [Designspace sampling](#7-designspace-sampling)
8. [Character and glyph coverage](#8-character-and-glyph-coverage)
9. [Rendering](#9-rendering)
10. [Performance](#10-performance)
11. [Privacy and licensing](#11-privacy-and-licensing)
12. [Tech stack and setup](#12-tech-stack-and-setup)
13. [Repository layout](#13-repository-layout)
14. [Milestone ladder](#14-milestone-ladder)
15. [Reference implementations](#15-reference-implementations)
16. [Testing](#16-testing)
17. [Stretch goals](#17-stretch-goals)
18. [References](#18-references)

---

## 1. Executive summary and scope

### The original statement

> Specimen tool for auditing variable axes, kerning pairs, and OpenType feature coverage across arbitrary sample text and weights.

Five findings reshape this:

1. **Browser text rendering cannot audit anything.** CSS plus `font-variation-settings` gives you pixels. It does not tell you which glyph IDs were produced, which lookups fired, whether a feature actually changed anything, or what a kern value was. **A specimen renders; an inspector needs the shaping internals.** See section 3.1.
2. **`shapeWithTrace` is the foundation, and it's what makes this a real tool.** harfbuzzjs exposes HarfBuzz's internal shaping trace — which lookup, in which feature, at which phase, transformed which glyphs. That turns "the output differs" into "GSUB lookup 12 in `liga` replaced glyphs 3 and 4 with glyph 87." Almost no browser-based font tool does this. See section 3.2.
3. **"Enumerate all kerning pairs" is combinatorially infeasible, and most people asking for it don't know why.** Modern kerning lives in GPOS `PairPos` format 2, which is class-based. A 40×40 class matrix over 300 covered glyphs expands to tens of thousands of effective pairs from a 1,600-entry table. You cannot list them; you can only audit the ones your text reaches, or expand classes with explicit bounds. See section 5.2.
4. **In a variable font, kerning itself varies across the designspace.** GPOS value records can carry `VariationIndex` deltas into the `ItemVariationStore`. Auditing kerning at a single instance is incomplete, and **this is the most genuinely novel audit this tool can offer** — nobody checks whether a kern pair collapses at Light or collides at Black. See section 5.4.
5. **`avar` means linear axis sampling is not linear in design space.** The `avar` table remaps normalized coordinates non-linearly, so stepping `wght` from 400 to 700 in even numeric increments may cluster or spread unevenly in the actual design. A sampling strategy that ignores `avar` audits the wrong points. See section 4.3.

### Revised project statement

> A client-side font audit tool built on HarfBuzz: trace-driven feature analysis that reports which lookups fired and what they changed, kerning audit scoped to reachable pairs with designspace-varying values surfaced, `avar`-aware designspace sampling that tests between named instances rather than only at them, character and glyph coverage reporting, and STAT validation — with shaping and rendering from one engine so the specimen and the analysis cannot disagree.

### What designers actually need

Worth stating, because it shapes the feature list more than "audit variable axes" does:

| Question | Section |
|---|---|
| "Which OpenType features actually do anything in my font?" | §6 |
| "Is my kerning broken at Light? At Condensed Black?" | §5.4 |
| "Where does interpolation break between my masters?" | §7.3 |
| "Will this font's style menu look right in InDesign?" | §4.5 (STAT) |
| "What characters am I missing?" | §8 |
| "Why isn't this ligature firing?" | §3.2 (trace) |

That last one is the killer. "Why isn't `ffi` forming?" currently requires opening a font editor and reading lookup tables by hand. A tool that answers it in one click has earned its existence.

### Explicit non-goals

- **Not a font editor.** Read-only. No outline editing, no table writing.
- **Not a font converter or subsetter.** Adjacent, well-served by existing tools.
- **Not a webfont optimizer.** Different product.
- **Not a full shaping engine.** You use HarfBuzz. Reimplementing GSUB/GPOS contextual chaining, Arabic joining, and Indic reordering is a multi-year project that has already been done correctly once.
- **Not multi-script parity in v1.** Latin, Greek, and Cyrillic first. Arabic and Indic shaping audits are valuable but each needs script-specific domain knowledge in the reporting.

---

## 2. Reality check

### 2.1 Three ways to get glyphs, and only one can audit

| Approach | Gives you | Can audit? |
|---|---|---|
| **Browser rendering** (CSS, canvas `fillText`) | Pixels | **No.** No glyph IDs, no lookup information, no feature attribution. |
| **Table parsing alone** (opentype.js, fontkit) | Font structure | **Partially.** Tells you what's *in* the font, not what *happens* when text is shaped. |
| **HarfBuzz** (harfbuzzjs) | Glyph IDs, positions, clusters, **traces** | **Yes.** |

**You need HarfBuzz and a table reader, and they answer different questions.**

HarfBuzz tells you what happened: these glyphs, these positions, these lookups fired. A table reader tells you what exists: this font declares `dlig` with four lookups covering these glyphs. The interesting audits sit in the gap — a feature that exists but never fires, a kern class that covers glyphs your text never uses, a named instance whose coordinates don't match its name.

A tool built on only one of the two will produce confident, incomplete answers.

### 2.2 The failure modes

| Symptom | Cause |
|---|---|
| "No kerning found" on a font with thousands of pairs | Reading only the legacy `kern` table, ignoring GPOS |
| Kerning report is missing most pairs | Class-based `PairPos` format 2 not expanded (§5.2) |
| Kerning looks fine but breaks in use | Audited at Regular only; varies across designspace (§5.4) |
| Feature reported "present" but does nothing | Confusing table presence with actual effect (§6.1) |
| Feature reported "absent" but works in the browser | Applied under a different script/langsys than the one queried |
| Axis samples cluster oddly | `avar` ignored (§4.3) |
| Specimen looks different from the analysis | Two engines — browser renders, HarfBuzz analyzes (§9.1) |
| Interpolation bugs never surface | Only named instances sampled (§7.1) |
| Font shows wrong names in app menus | Broken or missing STAT table (§4.5) |
| Tab freezes on a big sweep | Thousands of shaped runs on the main thread (§10) |

---

## 3. The shaping foundation ★

### 3.1 Why the browser can't help

`font-variation-settings` and `font-feature-settings` will render your font correctly — Chrome and Firefox both use HarfBuzz internally. But the browser gives you no access to the result beyond pixels.

You cannot ask it: which glyph is at position 3? Did `liga` fire? What was the kern between these two glyphs? Which lookup produced this substitution?

For a specimen, pixels are enough. For an *inspector*, they're nothing.

### 3.2 `shapeWithTrace` ★

harfbuzzjs exposes HarfBuzz's shaping trace through `shapeWithTrace`, returning `TraceEntry` records with a `TracePhase`. This is the internal record of what the shaper did, step by step.

```ts
import { shapeWithTrace } from "harfbuzzjs";

const trace = shapeWithTrace(font, buffer, features, stopAt, stopPhase);

// Each entry records a lookup application and the buffer state around it.
for (const entry of trace) {
  console.log(entry.m);        // message: which lookup, which phase
  console.log(entry.t);        // buffer contents at that point
  console.log(entry.glyphs);   // glyph names if available
}
```

This is the difference between a specimen and an inspector. With the trace you can answer:

- **"Why isn't my ligature firing?"** — because lookup 7 in `ccmp` decomposed the input first and the `liga` lookup no longer matches.
- **"What changed this glyph?"** — lookup 12, feature `calt`, at GSUB phase.
- **"Is this feature reaching my text at all?"** — the trace shows whether its lookups were even attempted.

**Build the trace viewer early.** It's the feature that makes the tool worth opening, and it's cheap once shaping is wired up. A collapsible list of lookups with before/after glyph runs is genuinely more useful than most of what commercial font tools expose.

### 3.3 One engine for shaping and outlines ★

harfbuzzjs also provides glyph outline extraction (`SvgPathCommand`) with variations applied, and glyph metrics (`GlyphExtents`, `FontExtents`).

**Use it for both.** If HarfBuzz shapes and something else draws, the two can disagree — different variation application, different metric interpretation, different rounding. The specimen would then show something the analysis didn't measure, which is exactly the class of bug this tool exists to find in *other people's* fonts.

This is the same architectural rule as any preview/export system: one source of truth, two consumers.

### 3.4 Feature control

HarfBuzz applies a default feature set per script. For Latin that's roughly `ccmp`, `locl`, `rlig`, `liga`, `clig`, `calt`, `kern`, `mark`, `mkmk`, plus `rvrn` for variable fonts. Others — `dlig`, `smcp`, `onum`, `ss01`–`ss20` — are opt-in.

```ts
// Explicitly disable a feature to diff against the default.
const features = [
  { tag: "liga", value: 0, start: 0, end: -1 },
];
```

The `start`/`end` range fields matter for a subtle audit: a feature can apply to part of a run. Most tools ignore this; exposing per-cluster feature ranges is a small differentiator.

**Note `rvrn` specifically.** Required Variation Alternates substitutes glyphs based on the current variation coordinates — a glyph that only appears below a certain weight, for instance. It's variable-font-specific and almost never audited. Showing where `rvrn` swaps take effect across the designspace is a genuinely useful and near-unique feature.

---

## 4. Variable font mechanics

### 4.1 The tables

| Table | Contains | Audit value |
|---|---|---|
| **`fvar`** | Axes (tag, min/default/max, name), named instances | Foundational |
| **`avar`** | Non-linear axis remapping | §4.3 — affects sampling |
| **`gvar`** | Glyph outline deltas (TrueType) | Interpolation quality |
| `CFF2` | Outline deltas (PostScript) | As above |
| **`HVAR`** | Advance width deltas | Advances vary! §4.4 |
| `MVAR` | Font-wide metric deltas | Line height varies |
| **`STAT`** | Style attributes, axis value names | §4.5 — commonly broken |
| **`GDEF`** | `ItemVariationStore` for GPOS deltas | §5.4 — kerning varies |

### 4.2 Coordinate spaces

Three coordinate spaces, and confusing them is a common bug:

```
user coordinates      wght 400, wdth 100      what designers type
       ↓ clamp to [min, max], then normalize piecewise around default
normalized            [-1, 0, 1]              default is always 0
       ↓ avar segment mapping
final normalized      [-1, 0, 1]              what deltas actually use
```

Normalization is piecewise linear around the **default**, not the midpoint:

```ts
function normalize(value: number, axis: AxisInfo): number {
  const v = Math.max(axis.min, Math.min(axis.max, value));
  if (v === axis.default) return 0;
  return v < axis.default
    ? -(axis.default - v) / (axis.default - axis.min)
    :  (v - axis.default) / (axis.max - axis.default);
}
```

So an axis with min 100, default 400, max 900 has its normalized zero at 400 — meaning the negative half spans 300 user units and the positive half spans 500. **Stepping normalized coordinates evenly is not the same as stepping user coordinates evenly**, and the tool should be explicit about which it's showing.

### 4.3 `avar` breaks linear sampling ★

The `avar` table contains a piecewise-linear segment map per axis, remapping normalized coordinates to normalized coordinates. Designers use it to make an axis feel visually even when the underlying interpolation isn't.

Which means: **even steps in user coordinates can be uneven steps in design space, and vice versa.**

```ts
function applyAvar(normalized: number, segments: SegmentMap): number {
  // Piecewise-linear interpolation through the (from, to) pairs.
  for (let i = 0; i < segments.length - 1; i++) {
    const a = segments[i], b = segments[i + 1];
    if (normalized >= a.from && normalized <= b.from) {
      const t = (normalized - a.from) / (b.from - a.from);
      return a.to + t * (b.to - a.to);
    }
  }
  return normalized;
}
```

Practical consequences for the tool:

- **Show the `avar` curve.** A plot of user coordinate against final normalized coordinate makes the remapping visible, and an accidentally steep or flat segment is instantly obvious.
- **Offer both sampling modes** — even in user space (what a designer types) and even in normalized space (what the interpolation actually does) — and label which is active.
- A font with a pathological `avar` is a real bug class, and it's invisible without this view.

Note that `avar` version 2 adds axis interaction, where one axis's mapping depends on another's position. Support is uneven; detect it and say so rather than silently mis-sampling.

### 4.4 Advances vary

`HVAR` carries per-glyph advance-width deltas. Without it, a `glyf`-based variable font derives advances from phantom points, which works but is less precise; `CFF2` fonts require `HVAR`.

The audit value: **a missing or incomplete `HVAR` means advance widths may not interpolate as intended**, which shows up as spacing that drifts across the designspace. Worth reporting presence and coverage.

### 4.5 STAT is commonly broken ★

The `STAT` table is required for variable fonts and tells applications how to name and organize styles. Get it wrong and the font appears in InDesign or Word with missing, duplicated, or nonsensical style names — a bug that never shows up in the designer's own testing because they're working in a font editor.

Worth validating:

- **Every `fvar` axis has a corresponding STAT design axis**
- **Every named instance maps to a valid combination of STAT axis values**
- **Elidable names are set correctly** — "Regular" should typically be elided so you get "Bold" rather than "Bold Regular"
- **`AxisValue` formats are consistent** — format 1 (discrete), 2 (range with a nominal), 3 (linked, for the bold link), 4 (multi-axis)
- **Axis ordering** produces a sensible menu order
- **No gaps or overlaps** in format 2 ranges

A STAT validation report is a small amount of code and addresses a real, frequent, hard-to-self-diagnose problem.

---

## 5. The kerning audit ★

### 5.1 Two places kerning lives

| Source | Notes |
|---|---|
| Legacy `kern` table | Flat pair list. Mostly absent from modern fonts; ignored entirely by some shapers for OpenType fonts with GPOS. |
| **GPOS `kern` feature** | Where modern kerning lives. `PairPos` format 1 (explicit pairs) and format 2 (class-based), plus contextual positioning via `ChainContextPos`. |

Report both, and flag the case where a font has a `kern` table that will be ignored because GPOS is present — that's a real authoring mistake and produces "my kerning doesn't work" reports.

### 5.2 Class kerning is combinatorial ★

`PairPos` format 2 defines two class definitions and a matrix of values. The effective pair count is the product.

```
Coverage: 300 glyphs
ClassDef1: 40 classes
ClassDef2: 40 classes
Value matrix: 1,600 entries
Effective pairs: up to 300 × (glyphs in ClassDef2) ≈ tens of thousands
```

A typical well-kerned Latin font expands to somewhere between tens and hundreds of thousands of effective pairs from a few thousand table entries. **"Show me all kerning pairs" is not a list you can render.**

Three workable approaches, and the tool should offer all three:

**1. Reachable pairs (default).** Shape the user's sample text, walk adjacent glyph pairs, report the kern applied to each. This is what a designer usually actually wants: "is the kerning right in *this* text."

**2. Class matrix view.** Show the class structure itself — classes as glyph groups, the value matrix as a heatmap. Compact, and it's how the kerning was authored, so it maps to the designer's mental model.

**3. Bounded expansion.** Expand classes into explicit pairs, but only within a user-selected glyph subset (a Unicode range, a script, or a named group), with a hard cap and a clear message when it's hit.

**Never attempt unbounded expansion.** A 200,000-row table in a browser is a hung tab, and the information density is near zero anyway.

### 5.3 Extracting applied kerning from shaping

The reliable way to get the kern *actually applied* is to shape with `kern` on and off and diff the advances:

```ts
function appliedKerning(font: Font, text: string): KernPair[] {
  const withKern = shape(font, text, [{ tag: "kern", value: 1 }]);
  const noKern   = shape(font, text, [{ tag: "kern", value: 0 }]);

  const pairs: KernPair[] = [];
  for (let i = 0; i < withKern.length - 1; i++) {
    const delta = withKern[i].xAdvance - noKern[i].xAdvance;
    if (delta !== 0) {
      pairs.push({
        left: withKern[i].glyphId,
        right: withKern[i + 1].glyphId,
        units: delta,
        cluster: withKern[i].cluster,
      });
    }
  }
  return pairs;
}
```

This is more trustworthy than reading the tables, because it accounts for contextual positioning, feature interaction, and whatever the shaper actually decided. It is also the only way to catch kerning that *should* apply but doesn't because another lookup interfered.

Cross-check against the table reader: a pair that exists in GPOS but shows zero applied delta is worth flagging — usually it means a preceding substitution changed the glyphs.

### 5.4 Kerning varies across the designspace ★

This is the audit nobody else offers.

GPOS `ValueRecord`s can include device tables, and in a variable font those can be `VariationIndex` records (delta format `0x8000`) pointing into the `ItemVariationStore` in `GDEF`. **The kern value is then a function of the variation coordinates.**

So a pair kerned at −40 units at Regular might be −15 at Light and −70 at Black. That's intentional and correct — tighter weights need different spacing. But it's also where bugs hide: a delta with the wrong sign produces a pair that looks fine at the default and collides at one end of the axis.

```ts
interface KernAcrossDesignspace {
  left: number;
  right: number;
  samples: Array<{ coords: Record<string, number>; units: number }>;
  min: number;
  max: number;
  range: number;              // max - min
  signChanges: number;        // crossing zero is usually suspicious
  collisionRisk: boolean;     // derived from sidebearings + kern at each sample
}
```

Two derived signals worth computing:

**Sign changes.** A pair that kerns negative at one end and positive at the other is almost always a mistake, and it's easy to detect.

**Collision risk.** Combine the kern value with the glyphs' sidebearings at each sample to estimate the actual gap. A gap approaching zero or negative at some point in the designspace is a real spacing bug. This requires outlines, which §3.3 already gives you.

Present it as a small sparkline per pair across the primary axis. A designer scanning a hundred sparklines spots the anomalous one immediately — far faster than reading numbers.

---

## 6. The feature coverage audit ★

### 6.1 "Coverage" means three different things

| Question | How to answer |
|---|---|
| **1. Is the feature in the font?** | Read the GSUB/GPOS feature list |
| **2. Does it apply to my script and language?** | Walk script → langsys → feature index |
| **3. Does it actually change anything for my text?** | **Shape with it on and off, and diff** |

Only the third is the useful audit, and it's the one table inspection cannot answer. A font can declare `dlig` under `latn`/`dflt` with lookups that cover no glyph in your sample, and a naive report says "present" while nothing happens.

Report all three, distinctly labeled. A designer needs to know the difference between "you didn't write this feature," "you wrote it but registered it under the wrong script," and "it's registered correctly but your text has no `ffi`."

### 6.2 Feature diffing

```ts
interface FeatureReport {
  tag: string;
  declared: boolean;
  scripts: string[];
  langSystems: string[];
  lookupCount: number;
  appliesByDefault: boolean;      // in HarfBuzz's default set for this script
  affectsSampleText: boolean;     // ← the one that matters
  substitutions: GlyphChange[];
  positionChanges: PositionChange[];
  lookupsFired: number[];         // from the trace (§3.2)
}

function auditFeature(font: Font, text: string, tag: string): FeatureReport {
  const off = shape(font, text, [{ tag, value: 0 }]);
  const on  = shape(font, text, [{ tag, value: 1 }]);

  return {
    tag,
    affectsSampleText: !glyphRunsEqual(on, off),
    substitutions: diffGlyphs(off, on),
    positionChanges: diffPositions(off, on),
    lookupsFired: tracedLookupsFor(font, text, tag),   // §3.2
    // ... structural fields from the table reader
  };
}
```

The comparison must cover glyph IDs *and* positions — some features (`kern`, `mark`, `mkmk`, `cpsp`) only change positioning, and a glyph-ID-only diff reports them as inert.

### 6.3 A text sample that exercises features

A useful audit needs text that can trigger things. Ship curated samples per feature:

```ts
const PROBES: Record<string, string> = {
  liga: "ffi ffl fi fl ff",
  dlig: "ct st sp Th",
  smcp: "Small Caps Text",
  c2sc: "ALL CAPS TEXT",
  onum: "0123456789",
  tnum: "1111 0000 8888",
  frac: "1/2 3/4 22/7",
  sups: "1st 2nd 3rd x2",
  case: "¿¡ -- (H) [H] {H}",
  zero: "0 100 007",
  ordn: "1o 2a No",
  salt: "aeg",
  calt: "AVATAR To Yo",
  kern: "AV To Ta Wa LT VA rn",
  ss01: "abcdefghijklmnopqrstuvwxyz",
};
```

And offer a **"probe everything" mode**: run every declared feature against its probe text plus the user's sample, and produce a single table of what fires and what doesn't. That report — one page, every feature, green or grey — is probably the single most useful artifact the tool produces.

### 6.4 Script and language matter

A feature registered only under `latn`/`TRK ` (Turkish) won't fire for default Latin. This is a real and common misregistration, especially for `locl`.

Let users set script and language explicitly, and default to auto-detection from the sample text. Report the resolved script/langsys alongside the results, because "it didn't fire" and "it didn't fire *under the script you selected*" are different findings.

---

## 7. Designspace sampling ★

### 7.1 Named instances are the tested points

A designer tests at their named instances because that's what the font editor shows. **The bugs are between them.**

An interpolation error — a kink, a self-intersection, a contour that crosses — typically doesn't exist at the masters. It appears somewhere in the middle, where nobody looked.

### 7.2 A sampling strategy

```ts
function samplingPoints(fvar: FvarTable, mode: SampleMode): Coords[] {
  switch (mode) {
    case "named":
      return fvar.instances.map(i => i.coordinates);

    case "extrema":
      // Every corner of the designspace box: 2^n for n axes.
      return cartesianProduct(fvar.axes.map(a => [a.min, a.max]));

    case "grid": {
      // n axes × k steps = k^n samples. 4 axes at 5 steps = 625.
      const k = 5;
      return cartesianProduct(fvar.axes.map(a => linspace(a.min, a.max, k)));
    }

    case "midpoints":
      // Between consecutive named instances — where bugs actually live.
      return midpointsBetweenInstances(fvar.instances);
  }
}
```

**Warn about the combinatorics.** Four axes at five steps is 625 samples; at ten steps it's 10,000. Show the count before running and let the user reduce it.

**`midpoints` should be the default suggestion**, because it directly targets the untested region while staying small.

And remember §4.3 — offer sampling in user space or in `avar`-corrected normalized space, and label which.

### 7.3 Detecting interpolation problems

At each sample, extract outlines and check for structural issues:

```ts
interface OutlineCheck {
  glyphId: number;
  coords: Coords;
  selfIntersects: boolean;        // contours crossing themselves or each other
  contourCount: number;           // must match across the designspace
  pointCount: number;             // must match — mismatch means bad compatibility
  extremeDeviation: number;       // how far from the interpolated-linear expectation
  windingConsistent: boolean;     // direction flips cause fill errors
}
```

**Contour and point count mismatches are the clearest signal.** Variable font interpolation requires compatible masters — same number of contours, same number of points, same order. A font that fails this usually won't build, but partial incompatibilities can slip through and produce wildly wrong intermediate shapes.

**Self-intersection** at intermediate values is the classic interpolation bug: two masters both fine, the middle produces a shape where a curve crosses itself, which renders as a hole or a filled notch depending on the fill rule.

Render a contact sheet: every sampled instance of a glyph in a grid, with flagged ones highlighted. Visual scanning finds these faster than any metric, and the metrics tell you where to look.

### 7.4 Metric drift

Advances vary (§4.4). Plot advance width per glyph across the primary axis and look for non-monotonic behavior or sudden jumps. A glyph that gets narrower going from Regular to Bold is usually a mistake, and it's trivially detectable.

---

## 8. Character and glyph coverage

### 8.1 Two different audits

**Character coverage** — which Unicode codepoints does `cmap` map? This answers "can I set text in Polish?"

**Glyph coverage** — which glyphs exist in the font? This includes glyphs unreachable from `cmap` and only produced by substitution: ligatures, small caps, alternates, positional forms.

Both matter. A font can have complete character coverage and a small-caps feature that only covers half the alphabet.

### 8.2 Coverage reporting

```ts
interface CoverageReport {
  totalGlyphs: number;
  mappedCodepoints: number;
  unreachableGlyphs: number[];       // exist but no cmap entry, no substitution path
  byBlock: Array<{ block: string; covered: number; total: number }>;
  byLanguage: Array<{ lang: string; covered: boolean; missing: string[] }>;
  missingCommon: string[];           // quotes, dashes, currency — the usual gaps
}
```

**Language coverage is more useful than Unicode block coverage.** "You're missing 3 characters for Polish: ą ę ł" is actionable; "Latin Extended-A is 73% covered" is not. Use the Unicode CLDR exemplar character sets, or one of the published per-language character requirement lists.

**Flag the common omissions specifically.** Proper quotes (" " ' '), en and em dashes, the ellipsis, non-breaking space, and the currency symbols the font's likely markets need. These are the gaps that turn up after release.

### 8.3 Unreachable glyphs

A glyph in the font with no `cmap` entry and no substitution rule producing it is dead weight — it inflates the file and can't be accessed.

Finding them requires walking every GSUB lookup and collecting output glyph IDs, then subtracting from the full glyph set along with the `cmap` targets. It's a satisfying audit and occasionally finds real leftovers from the design process.

---

## 9. Rendering

### 9.1 Draw from the same source that analyzes

harfbuzzjs provides outlines with variations applied (§3.3). Use those, positioned by HarfBuzz's own advances and offsets.

The alternative — render with the browser and analyze with HarfBuzz — means the specimen and the report can disagree, and the tool's credibility depends on them not disagreeing.

```ts
function renderRun(ctx: CanvasRenderingContext2D, run: ShapedGlyph[],
                   font: Font, size: number, x: number, y: number): void {
  const scale = size / font.unitsPerEm;
  let penX = x, penY = y;

  for (const g of run) {
    const path = getGlyphPath(font, g.glyphId);   // variations already applied
    ctx.save();
    ctx.translate(penX + g.xOffset * scale, penY - g.yOffset * scale);
    ctx.scale(scale, -scale);                      // font Y-up → canvas Y-down
    ctx.fill(path);
    ctx.restore();
    penX += g.xAdvance * scale;
    penY -= g.yAdvance * scale;
  }
}
```

Note the Y flip. Font coordinates are Y-up from the baseline; canvas is Y-down from the top. Getting this wrong produces upside-down text, which at least fails loudly.

### 9.2 Inspection overlays

The specimen becomes an inspector when you can see the structure:

- **Glyph boundaries** and advance widths
- **Sidebearings** shaded
- **Baseline, x-height, cap-height, ascender, descender** from `OS/2` and `hhea` — and worth flagging when those two disagree, which is a common and consequential authoring bug
- **Kern values** annotated between pairs
- **Cluster boundaries** — essential for understanding complex-script shaping
- **Glyph names and IDs** on hover
- **Feature attribution** — which glyphs came from which lookup (from the trace)

That last one is the payoff from §3.2 and is worth the effort. Hovering a ligature and seeing "produced by GSUB lookup 7, feature `liga`" is the interaction that makes people recommend a tool.

### 9.3 Waterfalls and grids

The specimen views designers expect:

- **Waterfall** — one string at increasing sizes
- **Weight ramp** — one string across the `wght` axis
- **Axis grid** — two axes as a 2D grid of samples
- **Paragraph** — real text at reading size, the only way to judge rhythm and color
- **Glyph table** — every glyph, with the current instance applied

The paragraph view matters more than it looks. Spacing problems that are invisible in a display-size waterfall are obvious in a block of 9pt body text.

---

## 10. Performance

### 10.1 The arithmetic

A designspace sweep is a lot of shaping and a lot of outlines:

```
20 sample strings × 20 weight steps × ~40 glyphs = 16,000 glyph draws
4 axes × 5 steps = 625 instances × 200 glyphs    = 125,000 outline extractions
```

Shaping is fast — HarfBuzz will do thousands of runs per second. **Outline extraction and rendering are the cost.**

### 10.2 Cache with quantized keys

Glyph outlines depend on the variation coordinates, which are continuous. Cache keys need quantization:

```ts
function glyphCacheKey(glyphId: number, coords: Coords): string {
  // Quantize to 1/1000 of normalized range. Finer than any visible difference,
  // coarse enough that slider drags hit the cache.
  const q = Object.entries(coords)
    .sort(([a], [b]) => a.localeCompare(b))
    .map(([tag, v]) => `${tag}:${Math.round(v * 1000)}`)
    .join(",");
  return `${glyphId}@${q}`;
}
```

Sorting the axis tags matters — otherwise `{wght, wdth}` and `{wdth, wght}` produce different keys for identical coordinates.

Cache `Path2D` objects rather than path data; `Path2D` construction is a meaningful share of the cost and it's reusable across draws.

### 10.3 Workers

Designspace sweeps belong in a Web Worker. HarfBuzz's WASM module can be instantiated there, and the main thread stays responsive.

Two notes: transfer shaped results as typed arrays rather than object graphs, and instantiate a separate HarfBuzz module per worker — the WASM memory isn't shareable across workers without care.

Report progress. A 625-instance sweep takes real time, and a progress bar with a cancel button is the difference between "thinking" and "frozen."

### 10.4 Virtualize long lists

A glyph table for a large font is thousands of cells; a kerning report can be thousands of rows. Virtualize both, and render glyph cells lazily as they scroll into view.

---

## 11. Privacy and licensing

**Process fonts entirely client-side. Never upload them.**

Fonts are licensed assets, often expensive ones, and frequently under licenses that restrict redistribution. A tool that uploads a font to a server is asking users to transmit a licensed file to a third party — which many of them are contractually not permitted to do, and all of them should be uneasy about.

Client-side-only is also simpler: no storage, no retention policy, no breach surface. WASM HarfBuzz runs fine in the browser, and there is no part of this tool that needs a server.

**Say so prominently in the UI.** "Your font never leaves your browser" is a real feature and the first thing a type designer will want to know.

Related: the `OS/2` `fsType` embedding bits are worth *reporting* as part of the audit — designers want to know what their own font declares — but you're not embedding anything, so it's informational rather than a gate.

---

## 12. Tech stack and setup

| Layer | Choice | Why |
|---|---|---|
| **Shaping, outlines, metrics** | **harfbuzzjs** | The actual shaping engine, with `shapeWithTrace` |
| **Table structure** | Purpose-built binary reader | See below |
| **Language** | TypeScript | |
| **Rendering** | Canvas2D + `Path2D` | Sufficient, and simple |
| **Workers** | Native Web Workers | Sweeps off the main thread |
| **Font formats** | `fontverter` or similar for WOFF/WOFF2 → SFNT | HarfBuzz wants SFNT |
| **UI** | React or Svelte | |
| **Charts** | `uPlot` or hand-rolled SVG | Sparklines and axis curves |

**On the table reader.** harfbuzzjs is deliberately low-level and, as its own ecosystem notes, requires careful handling to move data across the WASM boundary. It gives you shaping, not table structure.

For the structural audits — walking GPOS lookup trees, reading `STAT` axis values, expanding class definitions, enumerating `fvar` instances — you want direct access to the byte layout. These tables are well specified and a focused reader for the dozen tables you audit is a very reasonable few thousand lines. Libraries like opentype.js and fontkit parse fonts well (opentype.js does read GPOS kerning, not just the legacy `kern` table), but a purpose-built reader gives you the structure rather than a convenience abstraction over it — and structure is what you're reporting.

**Get a corpus early.** Google Fonts' variable fonts are freely available, varied, and include some with known quirks. Also collect deliberately broken ones — missing STAT, incompatible masters, misregistered features — because those are what the tool exists to find, and you can't develop the reports without examples.

---

## 13. Repository layout

```
Variable-Font-Inspector/
├── README.md
├── docs/
│   ├── design.md                ← this document
│   ├── audits.md                ← what each report means, for users
│   └── opentype-notes.md        ← table layout notes for contributors
├── src/
│   ├── shape/
│   │   ├── harfbuzz.ts          ← module init, font/face lifecycle
│   │   ├── shape.ts
│   │   ├── trace.ts             ← ★ shapeWithTrace parsing (§3.2)
│   │   ├── outlines.ts          ← ★ same engine as shaping (§3.3)
│   │   └── features.ts
│   ├── tables/                  ← purpose-built readers
│   │   ├── reader.ts            ← binary primitives
│   │   ├── fvar.ts
│   │   ├── avar.ts              ← ★ the segment map (§4.3)
│   │   ├── stat.ts              ← ★ validation (§4.5)
│   │   ├── gpos.ts              ← PairPos 1 & 2, class expansion
│   │   ├── gsub.ts
│   │   ├── gdef.ts              ← ItemVariationStore for §5.4
│   │   ├── cmap.ts
│   │   └── os2.ts
│   ├── audit/
│   │   ├── kerning.ts           ← ★ reachable, class, designspace (§5)
│   │   ├── features.ts          ← ★ diffing (§6)
│   │   ├── designspace.ts       ← ★ sampling + outline checks (§7)
│   │   ├── coverage.ts          ← characters and glyphs (§8)
│   │   ├── stat.ts
│   │   └── report.ts            ← unified report model
│   ├── render/
│   │   ├── glyph.ts
│   │   ├── cache.ts             ← quantized keys (§10.2)
│   │   ├── overlays.ts
│   │   └── views/               ← waterfall, ramp, grid, paragraph
│   ├── workers/
│   └── ui/
├── probes/
│   └── features.json            ← per-feature probe text (§6.3)
└── tests/
    ├── fixtures/                ← known-good and known-broken fonts
    ├── shape/
    └── audit/
```

---

## 14. Milestone ladder

### M0 — Scope and audit definitions
**Est. 3 days**

Decide which audits ship, which scripts are supported in v1, and what each report claims. Write `docs/audits.md` first — the report semantics determine the implementation.

**Done when:** for each audit you can state exactly what a green result means and what it doesn't rule out.

---

### M1 — HarfBuzz integration ★
**Est. 1.5 weeks**

Module init, font and face lifecycle, shaping with features and variations, outline extraction, **trace parsing**.

**Build the trace viewer in this milestone**, not later. It's the tool's differentiator and it validates that the integration is correct — if the trace makes sense, shaping is wired up right.

**Done when:** you can shape a string, see the glyph run, and expand a list of every lookup that fired.

---

### M2 — Table reader
**Est. 2 weeks**

`fvar`, `avar`, `STAT`, `cmap`, `OS/2`, `GDEF`, and GSUB/GPOS structure walking.

**Done when:** a font's axes, instances, features, and scripts are enumerable and match what a reference tool reports.

---

### M3 — Rendering and specimen views
**Est. 2 weeks**

Glyph rendering from HarfBuzz outlines, the glyph cache, waterfall, weight ramp, paragraph, glyph table, inspection overlays.

**Done when:** the specimen is pleasant enough to use for its own sake, and overlays show metrics and cluster boundaries.

---

### M4 — Feature audit ★
**Est. 1.5 weeks**

Three-level coverage reporting, on/off diffing covering both glyphs and positions, probe text, the "probe everything" report, script/langsys selection.

**Done when:** a single page lists every declared feature with whether it fires, and hovering a changed glyph names the lookup responsible.

---

### M5 — Kerning audit ★
**Est. 2 weeks**

`kern` table and GPOS, reachable-pair extraction by diffing advances, class matrix view, bounded expansion with caps, cross-checking applied against declared.

**Done when:** a font with class kerning reports correctly without attempting to enumerate 200,000 pairs.

---

### M6 — Designspace audits ★
**Est. 2 weeks**

`avar`-aware sampling with both modes, the `avar` curve plot, midpoint sampling, outline compatibility checks, self-intersection detection, metric drift, and **kerning across the designspace with sparklines**.

This is the milestone that makes the tool distinctive. Everything before it is table stakes.

**Done when:** a font with a deliberately bad delta on one kern pair is flagged automatically.

---

### M7 — Coverage and STAT
**Est. 1 week**

Character coverage by language, glyph coverage, unreachable glyphs, STAT validation.

---

### M8 — Reports and export
**Est. 1 week**

A unified report model, PDF or HTML export, shareable results.

A generated audit report a designer can send to a client or attach to a release is a genuinely useful output and mostly falls out of work already done.

---

## 15. Reference implementations

### 15.1 Class expansion with a hard cap

```ts
interface ExpansionResult {
  pairs: KernPair[];
  truncated: boolean;
  estimatedTotal: number;
}

function expandClassKerning(
  pairPos: PairPosFormat2,
  subset: Set<number> | null,
  cap = 50_000
): ExpansionResult {
  const class1Glyphs = groupByClass(pairPos.coverage, pairPos.classDef1);
  const class2Glyphs = groupByClass(allGlyphsIn(pairPos.classDef2), pairPos.classDef2);

  // Estimate BEFORE expanding, so the UI can warn rather than hang.
  let estimated = 0;
  for (let c1 = 0; c1 < pairPos.class1Count; c1++) {
    for (let c2 = 0; c2 < pairPos.class2Count; c2++) {
      if (pairPos.values[c1][c2].xAdvance !== 0) {
        estimated += (class1Glyphs[c1]?.length ?? 0) * (class2Glyphs[c2]?.length ?? 0);
      }
    }
  }

  const pairs: KernPair[] = [];
  outer:
  for (let c1 = 0; c1 < pairPos.class1Count; c1++) {
    for (let c2 = 0; c2 < pairPos.class2Count; c2++) {
      const v = pairPos.values[c1][c2].xAdvance;
      if (v === 0) continue;                       // zero entries aren't kerns
      for (const l of class1Glyphs[c1] ?? []) {
        if (subset && !subset.has(l)) continue;
        for (const r of class2Glyphs[c2] ?? []) {
          if (subset && !subset.has(r)) continue;
          if (pairs.length >= cap) break outer;
          pairs.push({ left: l, right: r, units: v });
        }
      }
    }
  }
  return { pairs, truncated: pairs.length >= cap, estimatedTotal: estimated };
}
```

Estimating before expanding is the important part. It lets the UI say "this would produce 187,000 pairs — narrow the glyph subset" instead of freezing for thirty seconds and then showing an unusable table.

### 15.2 Kerning across the designspace

```ts
async function kernAcrossDesignspace(
  face: Face, pairs: Array<[number, number]>, samples: Coords[]
): Promise<KernAcrossDesignspace[]> {

  const results = new Map<string, KernAcrossDesignspace>();

  for (const coords of samples) {
    const font = createFontAt(face, coords);      // variations applied
    for (const [l, r] of pairs) {
      // Shape the pair directly and diff advances — catches contextual
      // positioning that reading the tables would miss (§5.3).
      const units = shapedKernBetween(font, l, r);
      const key = `${l}/${r}`;
      const e = results.get(key) ?? emptyEntry(l, r);
      e.samples.push({ coords, units });
      results.set(key, e);
    }
    font.destroy();                                // WASM memory is manual
  }

  for (const e of results.values()) {
    const vals = e.samples.map(s => s.units);
    e.min = Math.min(...vals);
    e.max = Math.max(...vals);
    e.range = e.max - e.min;
    // A pair that crosses zero across the designspace is nearly always a bug.
    e.signChanges = countSignChanges(vals);
  }
  return [...results.values()];
}
```

`font.destroy()` matters. harfbuzzjs holds objects in the WASM heap and they are not garbage collected — a sweep that creates 625 fonts without freeing them will exhaust the heap.

### 15.3 STAT validation

```ts
function validateStat(stat: StatTable | null, fvar: FvarTable): StatIssue[] {
  const issues: StatIssue[] = [];

  if (!stat) {
    issues.push({ severity: "error", code: "STAT_MISSING",
      message: "Variable fonts require a STAT table. Style names will be wrong in most applications." });
    return issues;
  }

  const statTags = new Set(stat.designAxes.map(a => a.tag));
  for (const axis of fvar.axes) {
    if (!statTags.has(axis.tag)) {
      issues.push({ severity: "error", code: "STAT_AXIS_MISSING",
        message: `fvar axis '${axis.tag}' has no STAT design axis entry.` });
    }
  }

  for (const inst of fvar.instances) {
    if (!resolvableFromStat(inst.coordinates, stat)) {
      issues.push({ severity: "warning", code: "INSTANCE_UNNAMED",
        message: `Named instance at ${fmt(inst.coordinates)} cannot be named from STAT axis values.` });
    }
  }

  const elidable = stat.axisValues.filter(v => v.flags & ELIDABLE_AXIS_VALUE_NAME);
  if (elidable.length === 0) {
    issues.push({ severity: "warning", code: "NO_ELIDABLE",
      message: "No elidable axis values. Style names may read 'Bold Regular' rather than 'Bold'." });
  }
  return issues;
}
```

---

## 16. Testing

### 16.1 A fixture corpus of known-broken fonts ★

The tool's value is finding problems, so the test corpus must contain problems:

| Fixture | Should be flagged |
|---|---|
| No STAT table | `STAT_MISSING` |
| `fvar` axis absent from STAT | `STAT_AXIS_MISSING` |
| Incompatible masters (point count mismatch) | Outline compatibility failure |
| Self-intersection at 50% interpolation | Interpolation warning at midpoint |
| A kern pair with an inverted delta | Sign change across designspace |
| `dlig` registered under the wrong script | Declared but not applying |
| A `kern` table alongside GPOS | Legacy table will be ignored |
| Unreachable glyphs | Coverage report |
| Missing Polish diacritics | Language coverage |

Build some of these by hand with `fonttools` — deliberately corrupting a good font is the fastest way to get a fixture with a known expected finding.

### 16.2 Cross-check against reference tools

Shape the same text with your pipeline and with `hb-shape` (the HarfBuzz CLI) and assert identical glyph IDs and positions. If they diverge, your integration is wrong, and everything downstream is untrustworthy.

Similarly, cross-check table parsing against `ttx` output from `fonttools` — parse a font both ways and compare the structures.

### 16.3 Property tests

- Normalization maps default to exactly 0, min to −1, max to +1
- `avar` application is monotonic (a non-monotonic segment map is itself a font bug worth flagging)
- Shaping the same text twice gives identical results
- Class expansion never exceeds the cap
- The glyph cache key is order-independent across axis tags

### 16.4 Memory

Sweeps create many HarfBuzz objects. Test that a 500-instance sweep completes without growing WASM heap usage monotonically — a leak here surfaces as a crash partway through a long audit, which is the worst time for it.

---

## 17. Stretch goals

| Feature | Effort | Value |
|---|---|---|
| **Font comparison / diff** | Medium | Two versions side by side: what changed between releases? Very high value for type designers. |
| **Arabic and Indic shaping audits** | Large | Needs script-specific reporting knowledge, but underserved |
| **COLR/CPAL and SVG-in-OT** | Medium | Color font inspection; harfbuzzjs already exposes the paint trees |
| **Hinting inspection** | Large | `gasp`, `prep`, `fpgm`, and rendering at small sizes |
| **Automated regression suite for a font project** | Medium | Run the audits in CI on every font build — turns the tool into infrastructure |
| **Designspace visualization** | Medium | A 2D map of the designspace with masters and instances plotted |
| **OpenType feature file export** | Medium | Reconstruct approximate `.fea` from GSUB/GPOS. Useful and impressive. |
| **Optical size validation** | Small | Does `opsz` actually change anything, and sensibly? |
| **Web font subsetting preview** | Small | Which glyphs survive a subset for given text |

**The CI regression suite is the strongest of these.** A type foundry that can run "did this build break any kerning across the designspace?" on every commit has something genuinely valuable, and it reuses the entire audit layer with a headless runner.

---

## 18. References

### Specifications

| Source | For |
|---|---|
| **OpenType specification** (Microsoft) — `fvar`, `avar`, `gvar`, `HVAR`, `MVAR`, `STAT` | §4 |
| OpenType spec — `GSUB`, `GPOS`, `GDEF` | §5, §6 |
| **OpenType Layout tag registry** — script, language, feature tags | §6.4 |
| OpenType `ItemVariationStore` | §5.4 — how kerning varies |
| Unicode CLDR exemplar characters | §8.2 — language coverage |

### Tools and libraries

- **harfbuzzjs** API documentation — `Face`, `Font`, `Buffer`, `Variation`, `shapeWithTrace`, `SvgPathCommand`
- **HarfBuzz** documentation, especially the shaping and tracing sections
- `hb-shape` and `hb-view` CLI — the reference implementations to test against (§16.2)
- **fonttools** / `ttx` — for building fixtures and cross-checking (§16.1)
- **opentype.js** and **fontkit** — mature JS parsers; opentype.js reads GPOS kerning, not just the legacy table
- `fontverter` / `subset-font` — WOFF and WOFF2 conversion

### Background

- **Microsoft's "Variable Fonts" overview** and the OpenType 1.8 announcement material
- **Behdad Esfahbod's** writing and talks on shaping — the clearest explanation of what a shaper does and doesn't do
- Google Fonts' variable font guidelines — practical axis and STAT conventions
- The `fonttools` variable font documentation — designspace, masters, and compatibility requirements
- Existing specimen tools (Wakamai Fondue, Font Gauntlet, Axis-Praxis) — worth studying for what they do and don't expose

---

## Appendix A — Decision record

| Decision | Rationale |
|---|---|
| **HarfBuzz for shaping, not the browser** | CSS gives pixels. An inspector needs glyph IDs, positions, and lookup attribution. |
| **`shapeWithTrace` as the foundation** | Turns "the output differs" into "lookup 12 in `liga` did this." It's the tool's differentiator and answers the most-asked question. |
| Build the trace viewer in M1, not later | It validates the integration and it's why people will open the tool |
| **One engine for shaping and outlines** | If the specimen and the analysis come from different code, they can disagree — the exact bug class this tool finds elsewhere |
| A table reader *and* HarfBuzz | They answer different questions: what exists versus what happens. The interesting audits live in the gap. |
| Purpose-built binary table reader | Structural audits need the byte layout, not a convenience abstraction over it |
| **Never attempt unbounded kern pair enumeration** | Class-based `PairPos` format 2 expands to tens or hundreds of thousands of pairs; a 200,000-row table is a hung tab with no information density |
| Estimate expansion size before expanding | Lets the UI warn instead of freezing |
| Reachable pairs as the default view | "Is the kerning right in *this* text" is what designers actually ask |
| **Extract applied kerning by diffing advances, not by reading tables** | Accounts for contextual positioning and feature interaction, and catches kerning that should apply but doesn't |
| **Audit kerning across the designspace** | GPOS values carry `VariationIndex` deltas — kerning varies, and a wrong-signed delta looks fine at default and collides at Black |
| Sign changes and collision risk as derived signals | Cheap to compute, and both are almost always real bugs |
| Sparklines per pair | A designer scans a hundred at once; numbers require reading each |
| **`avar`-aware sampling, with both modes offered and labeled** | Even steps in user coordinates are not even steps in design space |
| Plot the `avar` curve | A pathological segment map is invisible otherwise |
| **Midpoint sampling as the default suggestion** | Named instances are where the designer already looked; bugs live between them |
| Warn about sampling combinatorics before running | 4 axes × 10 steps is 10,000 instances |
| **Feature "coverage" reported at three levels** | Not written / registered under the wrong script / present but nothing to act on are three different bugs |
| Feature diffs compare positions as well as glyph IDs | `kern`, `mark`, and `mkmk` change nothing about glyph identity |
| Ship per-feature probe text and a "probe everything" report | One page showing every feature green or grey is the most useful single artifact |
| **STAT validation as a first-class audit** | Broken STAT produces wrong style menus, and never shows up in the designer's own testing |
| Language coverage over Unicode block coverage | "Missing ą ę ł for Polish" is actionable; "73% of Latin Extended-A" isn't |
| Outline compatibility and self-intersection checks | Interpolation bugs don't exist at the masters |
| **Client-side only; the font never leaves the browser** | Fonts are licensed assets many users are contractually barred from transmitting |
| Glyph cache keyed on quantized, tag-sorted coordinates | Continuous coordinates otherwise never hit the cache; unsorted tags produce duplicate keys |
| `font.destroy()` after every sampled instance | harfbuzzjs objects live in the WASM heap and are not garbage collected |
| Sweeps in a Web Worker with progress and cancel | A 625-instance sweep otherwise looks like a frozen tab |
| Fixture corpus of deliberately broken fonts | You cannot develop a problem-finding tool without problems |
| Cross-check shaping against `hb-shape` | If your glyph runs differ from the reference, everything downstream is untrustworthy |

---

## Appendix B — Quick reference card

```
SHAPING
  browser CSS        → pixels only, audits NOTHING
  table parsing      → what EXISTS in the font
  HarfBuzz           → what HAPPENS when shaped
  ★ you need the last two; the gap between them is where the bugs are
  ★ shapeWithTrace → which lookup, which feature, which phase
  outlines from the SAME engine as shaping, or they can disagree
  font.destroy() — WASM heap is not GC'd

VARIABLE COORDINATES
  user → normalize (piecewise around DEFAULT, not midpoint) → avar → deltas
  normalize: v<def ? −(def−v)/(def−min) : (v−def)/(max−def)
  ★ avar remaps non-linearly — even user steps ≠ even design steps
  offer both sampling modes and LABEL which is active
  avar2 adds axis interaction — detect and say so

KERNING
  legacy `kern` table  vs  GPOS `kern` feature ← modern fonts use GPOS
  flag a font with BOTH — the legacy table gets ignored
  ★ PairPos format 2 is CLASS-based → combinatorial
    40×40 classes over 300 glyphs ≈ tens of thousands of pairs
    NEVER enumerate unbounded; estimate first, cap, offer a subset
  get APPLIED kerning by shaping with kern on/off and diffing advances
  ★ GPOS ValueRecords carry VariationIndex → KERNING VARIES BY INSTANCE
    audit across the designspace · sign changes ≈ always a bug
    combine with sidebearings → collision risk

FEATURES — "coverage" is three questions
  1 is it in the font?          → feature list
  2 under my script/langsys?    → script → langsys → feature index
  3 does it change my text?     → SHAPE ON/OFF AND DIFF  ← the useful one
  diff glyph IDs AND positions (kern/mark/mkmk change only position)
  rvrn is variable-specific and almost never audited

DESIGNSPACE
  named instances = where the designer already looked
  ★ bugs live BETWEEN them → midpoint sampling by default
  4 axes × 5 steps = 625 · × 10 steps = 10,000 — warn before running
  check: contour count · point count · self-intersection · winding
         advance monotonicity

STAT — commonly broken, never self-diagnosed
  every fvar axis needs a STAT design axis
  every named instance must be nameable from STAT axis values
  elidable names or you get "Bold Regular"

PRIVACY
  client-side ONLY. fonts are licensed assets users may not transmit.
  say so prominently — it's a feature.
```
