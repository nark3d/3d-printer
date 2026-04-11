# Ironing deep dive — OrcaSlicer top-surface smoothing

> A complete reference for top-surface ironing in OrcaSlicer: what it is physically, how it's implemented, how each setting interacts, how to calibrate it, what fails and why, and what values to use per material.
>
> **Scope**: covers PLA in detail (with a calibrated table for our specific hardware) and provides a general framework + placeholder tables for ASA, PETG, ABS, and TPU. The general section applies to all materials; the material tables capture the deviations.
>
> Same format and research methodology as the other topical deep dives in this folder. See [`seams_deep_dive.md`](seams_deep_dive.md) and the quality tab doc ([`quality_PLA.md`](quality_PLA.md) § Ironing) for cross-references.

---

## Table of contents

- [1. What ironing is physically](#1-what-ironing-is-physically)
- [2. When to use ironing (and when not to)](#2-when-to-use-ironing-and-when-not-to)
- [3. How ironing is implemented in OrcaSlicer](#3-how-ironing-is-implemented-in-orcaslicer)
- [4. Setting-by-setting reference](#4-setting-by-setting-reference)
- [5. Interaction with other quality settings](#5-interaction-with-other-quality-settings)
- [6. Calibration workflow](#6-calibration-workflow)
- [7. Gotchas and failure modes](#7-gotchas-and-failure-modes)
- [8. Per-material settings — PLA (calibrated)](#8-per-material-settings--pla-calibrated)
- [9. Per-material settings — ASA, PETG, ABS, TPU](#9-per-material-settings--asa-petg-abs-tpu)
- [10. Our current values across process profiles](#10-our-current-values-across-process-profiles)
- [11. References](#11-references)

---

## 1. What ironing is physically

Ironing is a **second pass** of the nozzle over a just-printed top surface, run at **very low flow** and a **specific pattern**, with the goal of using the **heat and mass of the hot nozzle** to iron out ridges, fill voids, and leave a uniform finish [^prusa-ironing-intro].

**The physical mechanism has three parts:**

1. **Heat transfer** — the nozzle is at print temperature (~220°C for PLA). Running it over solidified-but-recently-hot top infill reheats the top micron to soften, re-flow slightly, and re-solidify with surface tension evening out ridges
2. **Mechanical flattening** — the nozzle's flat underside physically drags across surface peaks, pushing small amounts of molten plastic sideways into the valleys
3. **Gap filling** — the extruder pushes out a **small amount** of fresh material (typically 8-15% of normal flow) to fill voids that heat-and-drag alone can't close

**Think of it as a literal clothes iron** — heat + pressure + a tiny bit of fresh material (spray starch) to smooth wrinkles. This analogy is from the PrusaSlicer documentation which explicitly draws it [^prusa-ironing-intro].

**What ironing changes vs a regular top surface:**

| Feature | Regular top surface | Ironed top surface |
|---|---|---|
| **Layer-line ridges** | Visible at angle | Almost gone |
| **Small voids between lines** | Visible as shiny spots | Filled |
| **Gloss/matte** | Matches material | Slightly glossier (heat-reflow) |
| **Dimensional accuracy** | Accurate to slicer | Slight growth at edges (fuzz / bulge) |
| **Print time** | Baseline | +2-10 minutes per top surface |
| **Print time for very flat tops** | Baseline | Up to +25% on models with large flat tops |

---

## 2. When to use ironing (and when not to)

Ironing is an **opt-in** feature because it has costs (print time, edge fuzz, heat creep risk) as well as benefits (beautiful top surfaces). The decision is object-dependent.

### When ironing helps

| Use case | Why ironing helps |
|---|---|
| **Nameplates, signs, badges** | The whole point is a visible flat top — ironing transforms acceptable → beautiful [^prusa-blog-ironing] |
| **Lids, enclosure tops, boxes** | Large flat top surfaces show layer lines most obviously |
| **Logo plates, plaques** | Decorative surfaces where finish quality > print time |
| **Glue surfaces** (parts that will be bonded) | Flatter surface = smaller gap = stronger glue joint [^prusa-ironing-intro] |
| **Text-on-top prints** | Smooth baseline makes text more legible |
| **Light-reflecting prints** | Matte lighting kills the layer-line look; glossy ironed top looks "finished" |
| **Machined-looking mechanical parts** | Pairs well with scarf seams for a "not obviously FDM" look |

### When ironing hurts or doesn't help

| Anti-use case | Why ironing fails |
|---|---|
| **Organic / curved top surfaces** | Ironing only works on flat surfaces — the slicer still runs it but the result is irregular |
| **Prints with many small top features** | The nozzle lifts, travels, drops between each feature — accumulating time + edge fuzz |
| **Vase mode prints** | Vase mode has no "top surface" — ironing is silently skipped |
| **Very thin top surfaces (< 1 mm of solid)** | Needs 4-6 solid top layers below for the ironing pass to have something to iron against [^snapmaker-ironing] |
| **Wood-filled filaments** | The wood particles melt poorly; ironing drags them into visible streaks [^prusa-knowledge-ironing] |
| **TPU / flex filaments** | The extrusion pressure relaxation time is much longer; ironing flow calibration is fussy |
| **Prints where edge sharpness matters** | Ironing always fuzzes edges slightly due to nozzle overshoot [^prusa-blog-ironing] |
| **Very tall thin prints with a small top** | The thin top gets its heat capacity stolen by the ironing pass, warping the cross-section |

### When ironing is a net time saver (counterintuitive)

Sometimes ironing *saves* time because it lets you drop other quality settings:

- **Fewer top layers**: without ironing, a flawless top needs 5-6 top layers. With ironing, 3-4 is enough — the iron pass fills any imperfections. Net time can drop
- **Lower layer height not required**: at 0.2 mm layer height with ironing, you can get results that would otherwise require 0.12 mm layer height across the whole print — massive time savings on tall prints

---

## 3. How ironing is implemented in OrcaSlicer

OrcaSlicer's ironing is a **direct inheritance** from PrusaSlicer (which was itself inspired by Cura's ironing feature, see issue prusa3d/PrusaSlicer#458 [^prusa-pr-cura]). The implementation details:

### When in the print sequence does ironing fire?

After the regular top surface infill for a given layer is complete, **and before moving to the next layer**, the slicer inserts an additional extrusion path that covers the top surface following the ironing pattern at the ironing speed and flow. The nozzle is **still at the layer Z height** (not lifted) so it drags across the surface it just deposited.

### What counts as a "top surface"?

Any layer region where the layer above has no material. OrcaSlicer identifies these during the slicing pass. The `ironing_type` setting then decides which of those regions actually get ironed:

- **`top`** (Top Surfaces) — every region that qualifies as "top-facing" anywhere in the model
- **`topmost`** (Topmost Surface) — only the single highest Z surface in the model [^bambu-ironing-types]
- **`solid`** (All Solid Layers) — every solid-infill region, including internal solid infill layers. Rarely used — dramatically slower and usually no benefit [^bambu-ironing-types]

### How does the slicer calculate the ironing path?

For each top-surface region:

1. Compute the **inset polygon** (top-surface shape minus `ironing_inset` margin) — this is the area that'll actually get ironed
2. Fill the inset polygon with lines according to `ironing_pattern` (concentric spirals, or rectilinear parallel lines)
3. Set the spacing between lines to `ironing_spacing`
4. If `ironing_angle_fixed` is enabled, use `ironing_angle` as the absolute direction for rectilinear lines. Otherwise, offset the angle from the top-surface infill direction by `ironing_angle` degrees
5. Calculate the per-line extrusion amount as: `normal top-surface E-volume × ironing_flow %`
6. Emit the path at `ironing_speed`

### What does OrcaSlicer NOT do that Cura/others do?

- **Paint-on ironing** (selectively iron specific regions in the preview) — requested in OrcaSlicer discussion #1389 but not implemented [^orca-discussion-1389]
- **Ironing calibration test** (built-in generator) — requested in issue #5411, closed as "not planned" [^orca-calibration-issue]
- **Conditional per-layer patterns** (e.g. "rectilinear for rectangular tops, concentric for round tops") — discussed in #4997 but not available [^orca-conditional-patterns]

For paint-on ironing, users work around by using **height-range modifiers** on specific Z ranges to enable/disable ironing per-region.

---

## 4. Setting-by-setting reference

Every OrcaSlicer ironing parameter, what it does, valid range, and community-tested values.

### 4.1 Ironing type (`ironing_type`)

Controls **which layers** are eligible for ironing.

| Value | Variable | What it does | When to use |
|---|---|---|---|
| **No ironing** | `no ironing` | Ironing disabled entirely | Default — do not iron unless needed |
| **Top Surfaces** | `top` | All top-facing surfaces, at every Z where a top exists | **The most common choice** — irons every flat top in the model [^orca-wiki-ironing] |
| **Topmost Surface** | `topmost` | Only the single highest-Z top surface | When you only care about the "final" visible top and want to save time on intermediate top surfaces [^bambu-ironing-types] |
| **All Solid Layers** | `solid` | Every solid infill layer, including internal ones | **Almost never useful** — very slow and rarely improves quality. Can help with prints requiring internal smoothness (unusual) [^bambu-ironing-types] |

**Default**: No ironing. **Most prints with flat tops**: `top`. **Tall prints where only the final top matters**: `topmost`.

### 4.2 Ironing pattern (`ironing_pattern`)

Controls the **path geometry** of the ironing pass.

| Value | What it does | When to use |
|---|---|---|
| **Concentric** | Spiral paths from the outer edge inward (or vice versa), following the surface contour | **Best for rounded / organic tops** — the path naturally follows curves. Minimises edge overrun on circular shapes |
| **Rectilinear** | Parallel straight lines, like standard rectilinear infill | **Best for rectangular / flat tops** — more efficient coverage on angular shapes. Slightly faster. Better tiger-stripe pattern when combined with `ironing_angle_fixed` [^orca-wiki-ironing] |

**Default**: Concentric (OrcaSlicer inherits this from PrusaSlicer). **For our quality-first PLA setup**: Concentric is safe. Rectilinear + fixed angle can give better results on large flat tops — see section 4.7.

**Gotcha**: Concentric can cause a **layer shift artefact** on very small top surfaces where the spiral closing point lands near the perimeter. Reported in OrcaSlicer issue #3796 [^orca-issue-3796]. For small features, rectilinear is safer.

### 4.3 Ironing flow (`ironing_flow`)

The **amount of material extruded** during the ironing pass, as a percentage of the normal flow rate [^orca-wiki-ironing].

**Physical meaning**: the extruder continues to push filament during the ironing pass, but at a tiny fraction of normal. If normal top-surface flow puts down 1 mm³ of plastic per mm of travel, 10% ironing flow puts down 0.1 mm³ per mm. That 0.1 mm³ is "fresh plastic to fill gaps" — it's not enough to build a new layer, just enough to even out the existing surface.

**Community-tested ranges** [^prusa-ironing-intro] [^all3dp-prusaslicer-ironing] [^makerworld-ironing-test]:

| Flow % | Effect | Use case |
|---|---|---|
| **5%** | Very light — may leave shiny grooves where the ironing pass had nothing to fill with | Only for already near-perfect top surfaces |
| **8-10%** | **Sweet spot for standard PLA** at 0.2 mm layer height | PLA default |
| **10-12%** | Good for PLA with slight under-extrusion or lower layer heights | Our PLA profile uses **12%** |
| **15%** | Works well for PETG, ASA, matte PLA | Material-dependent (see section 8-9) |
| **20-25%** | Fine layer heights (0.12 mm) where the normal top volume is tiny | Our 0.12 mm Fine profile uses **21%** |
| **30-40%** | Heavy ironing — risks over-extrusion. Some users report good PETG results here [^community-ironing-settings] | Rare |
| **50%** | Extreme — almost always over-extrudes | Not recommended |

**Flow % is sensitive to layer height**: lower layer heights need *higher* ironing flow percentages because the baseline (normal) flow is already small, and 10% of a small number may not be enough to fill voids. Rule of thumb: if you drop layer height by 2×, increase ironing flow by ~1.5× [^snapmaker-ironing].

### 4.4 Ironing line spacing (`ironing_spacing`)

The **distance between adjacent ironing passes** [^orca-wiki-ironing].

**Physical meaning**: how much the ironing path overlaps itself. A smaller spacing means each surface spot gets ironed by more passes (denser coverage, smoother result, slower). A larger spacing means fewer passes per area (rougher, faster).

**Community recommendation**: **≤ nozzle diameter**. On a 0.4 mm nozzle:

| Spacing | Effect |
|---|---|
| **0.05 mm** | Extreme density — ironing takes forever. Almost no quality improvement over 0.1 mm |
| **0.1 mm** | **Sweet spot** — 4× coverage (0.4/0.1), smooth result, reasonable time. Our PLA profile uses 0.1 [^orca-wiki-ironing] |
| **0.15 mm** | OrcaSlicer default — ~2.7× coverage. Good compromise for PETG / matte PLA / fine layer prints |
| **0.2 mm** | 2× coverage. Visible rectilinear lines start showing through |
| **0.25 mm** | 1.6× coverage. Generally too sparse for quality |
| **≥ 0.4 mm** | No overlap — single-pass ironing. Not really ironing anymore |

**Default**: 0.15 mm. **Quality-first recommendation**: 0.1 mm. **Speed-first**: 0.2 mm.

### 4.5 Ironing inset (`ironing_inset`)

The **distance from the edge** of the top surface to keep the ironing pattern inside. Prevents over-extrusion at the perimeter and keeps ironed edges crisp [^orca-issue-ironing-inset].

**Physical meaning**: the ironing path stops this many mm short of the actual top-surface edge. Why? Because the nozzle is physically wider than the path line — if ironing ran all the way to the edge, the nozzle's flat underside would push material *past* the edge, creating a bulge or fuzz.

| Inset (mm) | Effect |
|---|---|
| **0** | Ironing runs all the way to the edge. Maximum coverage, maximum edge fuzz [^orca-discussion-ironing-inset] |
| **0.1-0.15** | Slight inset — invisible in most cases. Cleanest result |
| **0.2-0.25 mm** | Orca default for many profiles. Small visible ring of un-ironed edge on some features |
| **0.5+ mm** | Heavy inset. Only useful for large flat tops with critical edge geometry. Creates a visible "frame" around the ironed area |

**Default**: varies (~0.21 mm in base profiles). **Quality recommendation**: 0.15-0.2 mm. **Our Customised profile uses 0.2 mm** — a conservative choice that prevents edge fuzz at the cost of a tiny un-ironed ring.

### 4.6 Ironing angle (`ironing_angle`)

The **angle of the ironing lines** (rectilinear pattern only). Interacts with `ironing_angle_fixed`.

**Two modes**:

1. **Offset mode** (when `ironing_angle_fixed = false`): the value is an *offset from the solid infill direction*. If solid infill is at 45° and ironing angle is 45°, ironing runs at 45° + 45° = 90°. **The offset rotates with the infill angle** between layers
2. **Fixed mode** (when `ironing_angle_fixed = true`): the value is an *absolute angle*. If set to 90°, ironing always runs horizontally regardless of infill direction

**Why this matters**: the ironing direction vs the underlying infill direction produces different optical effects. When the ironing path is parallel to the infill below, you get "tiger striping" — alternating shiny and matte bands from the infill ridges showing through. When it's perpendicular or at a fixed angle, the stripes become uniform [^orca-wiki-ironing] [^orca-issue-angle].

### 4.7 Ironing angle fixed (`ironing_angle_fixed`)

Boolean toggle for the angle mode above.

**When to enable**: when you want a **uniform appearance across all top surfaces** regardless of underlying infill direction. This is the quality-first choice. The OrcaSlicer wiki and issue #10834 both explicitly recommend enabling this with a fixed angle of 0° or 90° [^orca-wiki-ironing] [^orca-issue-angle].

**When to leave disabled**: the default behaviour (offset rotation) works for Cura-style "each layer rotates the ironing angle" aesthetics. Usually not beneficial for PLA.

**Our recommended values**:

- `ironing_angle_fixed: true`
- `ironing_angle: 90` (absolute, so all top surfaces iron horizontally)

This matches our existing PLA, ASA, and 0.12 mm Fine profiles — all use `90° + fixed`.

### 4.8 Ironing speed (`ironing_speed`)

The **travel speed during the ironing pass**, in mm/s. Separate from top-surface speed.

**Physical meaning**: how fast the nozzle drags across the surface. Slower = more heat transferred per mm = smoother result (and more risk of heat creep if too slow). Faster = less heat transferred, but less risk of heat issues.

**Community-tested range** [^prusa-ironing-intro] [^prusa-blog-ironing] [^community-ironing-settings] [^makerworld-ironing-test-2]:

| Speed | Effect |
|---|---|
| **10 mm/s** | Very slow — maximum heat transfer, glossiest finish, **highest heat-creep risk**, very long print time |
| **15-20 mm/s** | Traditional "quality" ironing speed from Prusa defaults |
| **25-30 mm/s** | Good balance of quality and print time for most PLA |
| **40 mm/s** | **Our PLA profile uses 40** — fast but works because our fan is at 100% and hotend is Rapido 2 HF (high flow, lower heat-creep tendency) |
| **50 mm/s** | Our ASA profile uses 50 — ASA handles faster ironing better than PLA because its higher print temp is further above the glass transition |
| **60+ mm/s** | Risk of ironing lines visible as "streaks" rather than smoothed surface. Generally too fast |

**Gotcha**: OrcaSlicer issue #10584 documents that **the ironing_speed setting doesn't always appear in the UI** depending on the tab configuration — it lives under the Speed tab, not the Quality tab, which can be confusing. It does function correctly when set.

**Default**: ~20 mm/s. **Our PLA**: 40 mm/s. **Our ASA**: 50 mm/s.

---

## 5. Interaction with other quality settings

Ironing doesn't exist in a vacuum. Several other settings affect how well it works.

### 5.1 Top surface line width

Controls the width of the **non-ironing** top infill lines that ironing then smooths over.

- Wider top surface lines (100-110% of nozzle) → fewer lines per area → fewer valleys between them → ironing has less work to do and produces better results
- Narrower top surface lines (< 95%) → more lines per area → more valleys → ironing can't fill all of them with 10% flow

**Recommended with ironing**: 100% (0.4 mm nozzle → 0.4 mm top line width). Our PLA profile is at 100%. See [`quality_PLA.md`](quality_PLA.md) § Line width.

### 5.2 Top surface pattern

Controls the **non-ironing** infill pattern that runs before the iron pass.

- **Monotonic** — unidirectional lines, best with ironing. Each line is in the same direction so ironing glides smoothly over them [^orca-wiki-ironing]
- **Monotonic Line** — same but explicit about the line type
- **Concentric** — the old default. Iron pass still works but ironing over concentric rings is less uniform
- **Rectilinear** — back-and-forth. Slightly less uniform than monotonic when ironed

**Recommended with ironing**: Monotonic or Monotonic Line. **Do not use Hilbert Curve, Gyroid, or other fancy patterns** as the top surface under ironing — they don't lay flat enough for the iron to fix.

### 5.3 Number of top layers

Controls **how much solid material sits under the ironed layer**.

- 2 top layers → ironing has nothing underneath, prints through holes show
- 3-4 top layers → **minimum for ironing to work** at 0.2 mm layer height [^snapmaker-ironing]
- 5-6 top layers → safer, fills imperfections from infill below
- 7+ top layers → diminishing returns, wastes material

**Recommended with ironing**: **1.0-1.2 mm of solid top** (at 0.2 mm = 5-6 layers; at 0.12 mm = 8-10 layers; at 0.28 mm = 4 layers).

### 5.4 Only one wall on top (`only_one_wall_top`)

Enables a **single perimeter wall** on the topmost layers, leaving more room for top-surface infill.

**Why this matters for ironing**: a single wall on top means the ironed area is larger (no second/third wall taking up space). The ironed surface goes closer to the outer edge — more of the top shows as smoothly ironed.

**Gotcha**: single-wall top combined with aggressive ironing can produce a "lip" where the wall meets the ironed infill — visible on oblique lighting. Use `min_width_top_surface` threshold to only apply on surfaces wide enough that this lip is hidden.

**Recommended with ironing**: `only_one_wall_top: true`, default `min_width_top_surface` threshold. See [`quality_PLA.md`](quality_PLA.md) § Walls and surfaces.

### 5.5 Scarf seams interaction

Scarf seams (see [`seams_deep_dive.md`](seams_deep_dive.md)) and ironing don't conflict directly — they operate at different layers:

- Scarf seams affect the outer wall transitions
- Ironing affects top-surface infill

**But**: the `wipe_on_loops` setting (which is off for scarf seams) can affect ironing quality because wipe-on-loops helps clean the nozzle before the ironing pass starts. With it off, any blob on the nozzle gets dragged into the first ironing pass.

**Recommended combination**: `wipe_on_loops: false` (for scarf seams) + **longer travel before ironing starts** (so any blob has a chance to drop off on travel) + ironing active. This combination works in our profile.

### 5.6 Ironing and pressure advance

The ironing pass uses a **very low flow**, which means pressure advance compensation has less effect. BUT:

- If PA is miscalibrated *too high*, the slicer will under-extrude the start of each ironing line, leaving gaps at the start of every line
- If PA is too low, overshoot blobs at line starts

**Recommended**: calibrate PA properly first (see [`seam_diagnosis_field_notes.md`](seam_diagnosis_field_notes.md) for per-filament PA values), **then** tune ironing. Don't try to tune both at once.

---

## 6. Calibration workflow

The correct order of operations to calibrate ironing **on a new filament or a new printer**:

### Step 0 — Prerequisites

Before touching ironing, confirm:

1. **Flow ratio is calibrated** — if top surfaces are over- or under-extruded, ironing can't fix that. See OrcaSlicer's Flow Rate calibration.
2. **Pressure advance is calibrated** — bad PA shows at ironing line starts.
3. **Temperature is in the right range** — too cold means ironing can't reflow the surface; too hot means the nozzle drags melted plastic everywhere.
4. **First-layer and top-surface settings are good** — layer height, top layer count, top surface pattern (Monotonic).

### Step 1 — Print a baseline without ironing

Slice and print a simple test object (50 × 50 × 5 mm cube, flat top) **without ironing enabled**. Document:

- How flat is the top surface by eye?
- Are there visible layer lines, voids, or stringing on the top?
- Is over- or under-extrusion visible?

If the baseline is bad, **fix flow/PA/line width first**. Ironing won't save a bad baseline.

### Step 2 — Enable ironing with default settings

Set:

- `ironing_type: top`
- `ironing_pattern: Concentric` (start safe)
- `ironing_flow: 10%`
- `ironing_spacing: 0.1 mm`
- `ironing_speed: 25 mm/s`
- `ironing_inset: 0.15 mm`
- `ironing_angle_fixed: true`
- `ironing_angle: 90`

Print the same test cube. Observe:

- **Better than baseline?** → good, proceed to fine tuning
- **Worse (bulging, edge fuzz, drag marks)?** → flow is too high. Drop to 8%
- **No improvement (still shows layer lines)?** → flow is too low or spacing too wide. Bump flow to 12% and/or tighten spacing to 0.08 mm

### Step 3 — Run a flow calibration grid

Print an ironing calibration model from MakerWorld [^makerworld-ironing-test] [^makerworld-ironing-test-2] with varying flow % and speed combinations in a 3×3 or 5×5 grid. Common test grids:

- **Flow**: 5%, 10%, 15%, 20%, 25%
- **Speed**: 15, 25, 35, 50 mm/s

Look for the cell that's **smoothest without bulging**. Note it down — that's your material's sweet spot for this printer.

### Step 4 — Visual inspection under angled light

Ironed surfaces reveal their flaws under **raking light** (a desk lamp at a low angle). The telltale signs:

| What you see | What it means | Fix |
|---|---|---|
| **Visible tiger stripes** (alternating shiny/matte bands) | Ironing angle parallel to infill | Set `ironing_angle_fixed: true`, angle 90° |
| **Uniform shiny surface** | Ironing is working | Stop — you're done |
| **Raised ridges at ironed edges** | Flow too high OR inset too low | Drop flow 2%, increase inset to 0.2 mm |
| **Shiny grooves between lines** | Flow too low OR spacing too wide | Raise flow 2%, tighten spacing to 0.08 mm |
| **Drag marks / scraped patches** | Z height is too close (first layer too low, and every subsequent layer too) | Not an ironing problem — fix Z-offset |
| **Discoloured patches** | Hotend temp too high for the ironing dwell time | Drop print temp 5°C |

### Step 5 — Document and save

Once you hit the sweet spot, save the values and document the *why*. Ironing is sensitive enough that it drifts with filament changes, so note which filament you calibrated on.

---

## 7. Gotchas and failure modes

### Gotcha 1 — Heat creep during ironing

**The single biggest risk with ironing.** Extrusion during ironing is very slow (low flow, low speed = low mm³/s). This means filament sits in the hot end longer than normal, and the heat from the hotend can creep up the filament path, softening it above the heatsink and causing a clog [^prusa-knowledge-ironing] [^heat-creep-general].

**Why PLA is worst**: PLA has the lowest softening temperature (~60°C glass transition). During ironing at 25 mm/s with 10% flow, the filament dwell time in the hot zone doubles or triples.

**Symptoms**:
- Extrusion gradually decreases mid-ironing pass
- Audible click from the extruder (skipping)
- Filament is visibly blob-swollen when inspected after the failed print
- The clog clears on cooldown + manual unload

**Prevention**:
- **Ensure hot end cooling fan is at 100%** (our setup already does this)
- **Keep ambient temperature below 30°C** (matters for enclosed chambers)
- **Don't iron too slow** — anything below 15 mm/s is asking for trouble on PLA
- **Don't drop print temperature during ironing** — if the temperature drops because of slow flow, heat creep gets worse
- **Increase retraction length temporarily** if you must iron slowly (but note: matte PLA already calibrated for low retraction)

**Our PLA profile uses 40 mm/s ironing speed** specifically to avoid heat creep on long ironing passes. This is a deliberate trade-off against the pure-quality 20 mm/s default.

### Gotcha 2 — Edge fuzz

The **nozzle is wider than the ironing line**. The flat underside of the nozzle extends ~0.4 mm around the extrusion point. When ironing runs close to a top-surface edge, the nozzle's wide underside pushes molten plastic past the edge, creating a fuzzy or bulged ring [^prusa-blog-ironing] [^prusa-knowledge-ironing].

**Visible as**: a soft, rounded edge on what should be a sharp edge. More obvious on small flat tops (< 20 × 20 mm).

**Fix**: increase `ironing_inset` to 0.2-0.3 mm so the ironing path stays further from the edge.

### Gotcha 3 — Ironing marks on small multi-top prints

When a print has multiple small flat tops at different Z heights, the ironing pass runs on each one. The nozzle has to lift, travel, drop, iron, lift, travel, drop, iron... for every small top. This creates:

- **Blob on every restart** as the extruder re-primes after the lift
- **Ooze during travel** that drops on the surface
- **Additional ironing-direction "start marks"** where the path begins
- **Drag marks where the nozzle lands**

Reported in Bambu forum "Top layer small areas with marks" [^bambu-forum-small-areas].

**Fix options**:

1. **Disable ironing for this specific print** (it's not worth the hassle)
2. **Use `ironing_type: topmost`** so only the single highest top gets ironed
3. **Increase retraction** to reduce travel ooze
4. **Lower ironing flow** to minimise the blob at each restart
5. **Enable `wipe_on_loops` temporarily** for this print (wipes the nozzle before each ironing pass starts, reducing travel ooze — but this conflicts with scarf seams, so use a separate profile)

### Gotcha 4 — Edge "ghosting" ring

On ironed rectangular tops, some users see a faint rectangular "ghost" around the ironed area about 0.1-0.2 mm inside the edge [^bambu-forum-ironing-edges]. This is caused by the ironing path's **perimeter-following inset** — the ironing pattern is a polygon inside the top-surface polygon, and the transition between the un-ironed 0.2 mm strip and the ironed inner area shows up as a faint step.

**Fix**: reduce `ironing_inset` to 0 or 0.05 mm (accept slight edge fuzz in exchange for no ghost) OR increase it to 0.3+ mm (so the inset is deliberate and looks like a frame).

### Gotcha 5 — Only one pass on layer X, missing on layer Y

The ironing logic depends on **what the layer above looks like**. If the model has weird geometry (a tall thin feature poking out of a flat top), the flat top might be classified as "solid infill with thin exception" rather than "top surface", and ironing silently skips it.

**Symptom**: ironing works on one test print but silently doesn't apply on a similar one. The preview in OrcaSlicer shows ironing paths on one and not on the other.

**Fix**: check the preview slider — zoom to the layer you expect to be ironed and look for the ironing paths. If not present, the slicer didn't classify that layer as "top". Sometimes adjusting `min_width_top_surface` or `only_one_wall_top` helps.

### Gotcha 6 — Surface gloss mismatch

Ironed surfaces are **slightly glossier than non-ironed surfaces of the same filament**, even matte PLA. This is because the iron pass re-melts the top micron and lets surface tension flatten it. On a **partially ironed model** (e.g. large flat top + smaller tilted tops that don't get ironed), the tilted tops remain matte while the flat tops are glossy — creating a visible "finish mismatch" [^prusa-forum-ironing-examples].

**Fix options**:

1. **Accept it** — some people like the contrast
2. **Use a glossy filament** — the mismatch is hidden when everything's already glossy
3. **Reduce ironing flow below normal** — less melting, less gloss transfer
4. **Use non-ironed Monotonic top pattern** — gives a uniform finish across flat and tilted tops

Matte PLA is the worst affected — see [`../filament/sunlu_pla_matte.md`](../filament/sunlu_pla_matte.md) § gotchas for more context on matte-specific surface gloss issues.

### Gotcha 7 — Dimensional bleed

Aggressive ironing can cause the top few microns of the print to be **slightly larger** than the sliced dimensions because of the edge nozzle-push. On precision mechanical parts this matters — a 50.00 mm feature might come out at 50.10 mm.

**Fix**: use `ironing_inset: 0.3-0.5 mm` on precision prints to keep the ironing away from critical dimensions. Or disable ironing entirely for dimensional parts.

### Gotcha 8 — First layer of a tall thin print

If the "top surface" is a very thin layer above a narrow cross-section (e.g. the cap of a thin pillar), ironing the top will **heat the pillar below** and cause warping / lean. Rare but dramatic when it happens.

**Fix**: use `ironing_type: topmost` only, and only on models where the topmost surface is adequately supported.

### Gotcha 9 — Ironing the wrong thing

Some users enable `ironing_type: solid` thinking it's "the best quality option" — it isn't. It irons **internal solid infill layers** (layers 1, 2, 3 in a solid section) which are **covered by walls** and not visible. The result is +50-100% print time for zero visible benefit [^bambu-ironing-types].

**Fix**: always use `top` or `topmost`, never `solid` unless you have a very specific reason.

---

## 8. Per-material settings — PLA (calibrated)

**Our hardware**: Rapido 2 HF + Orbiter 2.5 + 0.4 mm hardened steel nozzle, Klipper-driven. See [`../printer/README.md`](../printer/README.md) for full context.

**Profile**: `0.20mm Standard @MyKlipper - PLA` (our main PLA process profile).

### PLA recommended values

> **Updated 2026-04-11**: pattern changed to Rectilinear and inset set to explicit 0.15 mm. See [`../changelog/2026-04-11-ironing-tune.md`](../changelog/2026-04-11-ironing-tune.md) for the rationale.

| Setting | OrcaSlicer default | Our value | Rationale |
|---|---|---|---|
| **Type** (`ironing_type`) | no ironing | **`top`** | Iron all top surfaces. Use `topmost` only for tall prints where intermediate tops are structural, not visual |
| **Pattern** (`ironing_pattern`) | Concentric | **`rectilinear`** | Changed 2026-04-11. Rectilinear avoids Concentric's "center dot" artefact and small-top layer-shift (Issue #3796). Pairs optimally with our fixed 90° angle. More versatile for the flat-rectangular tops on most of our prints |
| **Flow** (`ironing_flow`) | 10% | **12%** | Slightly above default — our PLA flow ratio is 0.98 (slightly under), and 12% compensates without bulging [^orca-wiki-ironing] [^all3dp-prusaslicer-ironing] |
| **Line spacing** (`ironing_spacing`) | 0.15 mm | **0.10 mm** | Tighter than default for quality. ~4× coverage on 0.4 mm nozzle. Nominal time penalty |
| **Inset** (`ironing_inset`) | ~0.21 mm | **`0.15 mm`** explicit | Changed 2026-04-11. Community quality value (deep dive § 4.5 "cleanest result" range). Crisper edges and better-defined corners without edge fuzz — 0.06 mm closer to edge than inherited default |
| **Angle** (`ironing_angle`) | -1 (auto rotation) | **90°** | Absolute horizontal lines across all top surfaces — uniform finish, reduces tiger striping [^orca-issue-angle] |
| **Fixed angle** (`ironing_angle_fixed`) | false | **true** | Enables the absolute 90° above. Required for the tiger-stripe fix to work |
| **Speed** (`ironing_speed`) | ~20 mm/s | **40 mm/s** | Faster than the Prusa-traditional 20 mm/s, to reduce heat creep risk on PLA. Our fan is at 100% and Rapido 2 HF tolerates this. Tested and works |
| **Top surface pattern** (`top_surface_pattern`) | inherited | **`monotonicline`** explicit | Changed 2026-04-11. Explicit pairing with ironing — ironing works best on monotonic/monotonicline top surfaces (deep dive § 5.2). Locks the value in place against future parent-profile changes |

### PLA-specific considerations

- **Matte PLA** (Sunlu PLA Matte etc): keep the same ironing settings, but watch for increased edge fuzz because matte additives change the melt surface tension. Consider reducing `ironing_flow` to 10% for pure aesthetic jobs
- **PLA+ / PLA Pro**: same settings. The added toughness doesn't affect ironing
- **Silk PLA**: **reduce flow to 8-10%** and **speed to 30 mm/s** — silk additives make the melt more fluid, so less fresh material is needed to fill voids
- **Wood-filled PLA**: **do not iron** — the wood particles create visible streaks [^prusa-knowledge-ironing]
- **Glow-in-the-dark PLA**: mildly abrasive, but ironing doesn't exacerbate this. Normal PLA settings work
- **Translucent PLA**: ironing creates a visible "shine stripe" pattern that reads as a watermark. Either disable ironing or embrace the effect

---

## 9. Per-material settings — ASA, PETG, ABS, TPU

The starting values below are based on community consensus [^prusa-knowledge-ironing] [^prusa-ironing-intro] [^community-ironing-settings]. **These are starting points — always calibrate per filament.**

### ASA

**Note**: ASA **irons exceptionally well** [^prusa-knowledge-ironing]. Top surfaces come out smoother than any other common FDM material.

| Setting | Our value | Rationale |
|---|---|---|
| **Type** | `top` | Same as PLA |
| **Pattern** | Concentric | Safe default |
| **Flow** | **15%** | ASA needs slightly more fresh material than PLA. Our ASA profile uses 15% |
| **Spacing** | **0.10 mm** | Same as PLA |
| **Inset** | default | Same |
| **Angle** | **90°** fixed | Same tiger-stripe reduction strategy |
| **Speed** | **50 mm/s** | ASA prints at higher temps, tolerates faster ironing without heat creep. Our ASA profile uses 50 mm/s |

**ASA gotchas**:
- Print temp should be 240-250°C; ironing at the same temp is fine
- Enclosed chamber is required for ASA — good for warping but bad for heat creep. Monitor extruder cold-side temp
- ASA can develop surface cracks if ironed too aggressively — start at 10% flow and work up

### PETG

**Note**: PETG irons **adequately but not brilliantly** [^prusa-knowledge-ironing]. The main issue is nozzle-wiping — PETG likes to stick to the nozzle and get dragged into the ironing path.

| Setting | Placeholder value | Rationale |
|---|---|---|
| **Type** | `top` | Same |
| **Pattern** | Concentric or Rectilinear | Either works |
| **Flow** | **15%** (starting) — tune 12-20% | PETG's higher viscosity needs more fresh material to fill gaps |
| **Spacing** | **0.12 mm** | Slightly wider than PLA to reduce the nozzle-drag-blob problem |
| **Inset** | **0.25 mm** | Larger than PLA to keep blob-on-edge away from edges |
| **Angle** | **90°** fixed | Same |
| **Speed** | **30-35 mm/s** | Slower than PLA to give more time for the melt to settle |

**PETG gotchas**:
- **Wipe the nozzle** before every ironing pass — PETG ooze is the #1 failure mode
- **Consider using a silicone nozzle sock** if available
- **Print temp should be 230-245°C**
- Some users report **35% flow** works best for PETG — this is unusually high but reflects PETG's viscosity [^community-ironing-settings]

### ABS

| Setting | Placeholder value | Rationale |
|---|---|---|
| **Type** | `top` | Same |
| **Pattern** | Concentric | Safe default |
| **Flow** | **15%** (starting) — tune 10-18% | Similar to PETG |
| **Spacing** | **0.12 mm** | Slightly wider than PLA |
| **Inset** | **0.25 mm** | Same as PETG |
| **Angle** | **90°** fixed | Same |
| **Speed** | **30-40 mm/s** | Between PLA and ASA |

**ABS gotchas**:
- **Enclosed chamber required** — fans during ironing will cause warping
- **Reduce cooling fan to 20-30% during ironing** — ABS is sensitive to airflow-induced shrinkage
- **Temperature control is critical** — ABS ironing failures during testing are often blamed on temperature fluctuations [^snapmaker-ironing]

### TPU / Flex

**Note**: TPU is the **worst material for ironing**. The long pressure-relaxation time of TPU melt means the flow setting is extremely sensitive — a little too high and it bulges, a little too low and it leaves gaps. Most users disable ironing on TPU.

| Setting | Placeholder value | Rationale |
|---|---|---|
| **Type** | `no ironing` (usually) | TPU is not worth the hassle |
| **If you must iron** | Flow 25-30%, Speed 15-20 mm/s, Spacing 0.15 mm | |

**TPU gotchas**:
- Pressure advance must be extremely well calibrated first
- Heat creep risk is lower than PLA because TPU print temps are higher
- Edge fuzz is much worse because TPU doesn't rigidly hold its edge shape

### ABS/ASA extended and PLA-CF (carbon fibre)

- **PLA-CF / Carbon-fibre filled**: ironing IS possible but the fibres create visible streaks. Reduce flow to 8% and accept the streaks, or disable entirely
- **PA/PA-CF (Nylon)**: ironing is temperature-sensitive because nylon's melt pool is much thinner. Use Prusa-recommended 15% flow / 20 mm/s as a starting point

---

## 10. Our current values across process profiles

> **Updated 2026-04-11 after the ironing tuning pass.** The PLA profile was updated with explicit `rectilinear` pattern and `0.15 mm` inset. See [`../changelog/2026-04-11-ironing-tune.md`](../changelog/2026-04-11-ironing-tune.md).

| Profile | Type | Pattern | Flow | Spacing | Inset | Angle | Fixed | Speed |
|---|---|---|---|---|---|---|---|---|
| `0.20mm Standard @MyKlipper - PLA` ⭐ | `top` | **`rectilinear`** | **12%** | **0.10** | **`0.15`** | **90°** | **✓** | **40 mm/s** |
| `0.20mm Standard @MyKlipper - Clean ASA Plus` | `top` | (default Concentric) | **15%** | **0.10** | default | **90°** | **✓** | **50 mm/s** |
| `0.12mm Fine @MyKlipper - Customised` | `top` | (default Concentric) | **21%** | **0.10** | default | **90°** | **✓** | (inherits) |
| `0.20mm Standard @MyKlipper - Customised` | `top` | (default Concentric) | **8%** | **0.18** | **0.2** | **135°** | (default) | **40 mm/s** |

⭐ = fully tuned per the deep dive's "best possible PLA" recommendation as of 2026-04-11.

### Observations

1. **The PLA profile is now fully tuned** — explicit rectilinear pattern, explicit 0.15 mm inset, paired with `top_surface_pattern: monotonicline` (also made explicit). This is the deep dive's "cleanest result" PLA configuration
2. **ASA, 0.12mm Fine profiles** are aligned on the core quality settings (fixed 90° angle, tight 0.10 mm spacing, `top` type) but still use the default Concentric pattern. **Consider applying the same rectilinear + 0.15 inset treatment** to these profiles if you use them for quality-first flat-top prints
3. **The `Customised` profile is still different** — it uses `ironing_angle: 135` (no `fixed` flag), flow 8%, spacing 0.18, inset 0.2. Looks like an older profile that hasn't been updated. **Recommend deprecating or bringing in line with the others**
4. **The 0.12mm Fine profile has 21% flow** — defensible because at 0.12 mm layer height, the base flow is ~60% of 0.2 mm flow, so the ironing needs a proportionally higher percentage to fill voids. **Validates the "lower layer → higher flow %" rule**

### Remaining recommendations for our profiles

| Change | Profile | Rationale | Priority |
|---|---|---|---|
| **Apply ironing tuning to ASA profile** | 0.20mm Clean ASA Plus | Same treatment as the PLA profile: explicit `rectilinear` + `0.15 mm` inset. ASA profiles benefit from the same pattern + inset logic | Medium — do when you next print ASA |
| **Deprecate or update `Customised` profile** | 0.20mm Customised | Drifted settings (135° no-fixed, 8% flow, 0.18 spacing). Either update to match PLA or delete | Low — only matters if you actually use this profile |
| **Add explicit `ironing_speed` to 0.12mm Fine profile** | 0.12mm Fine | Currently inherits. Suggested: 30 mm/s (slower than our 0.2 mm profile because the 0.12 mm layer is more heat-sensitive) | Low — only matters when you print at 0.12 mm |
| **Per-filament ironing flow for matte PLA** | Sunlu PLA Matte (filament profile) | Community findings suggest matte PLA wants 15-20% flow vs the 12% process default. Set `filament_ironing_flow` on the Sunlu profile after running an ironing calibration test to validate | Medium — do when you next dial in Sunlu Matte |

None of these are urgent. The PLA profile is now fully tuned.

---

## 11. References

[^prusa-ironing-intro]: Prusa Knowledge Base — Ironing. Describes ironing as "running a special second infill phase in the same layer" where "the hot nozzle travels over the just printed top layer, flattens any plastic that might have curled up" and "extrudes a small amount of filament to fill in any holes". Also the authoritative "clothes iron" analogy. Notes that PLA "irons very nicely, however, it is most prone to heat creep", PETG "irons well, but there's an increased risk of extra filament sticking to the nozzle", ASA "irons incredibly well, producing super smooth top surfaces", and wood-filled filaments give poor results. <https://help.prusa3d.com/article/ironing_177488>

[^prusa-blog-ironing]: Prusa Blog — PrusaSlicer 2.3 Ironing Guide. Details the "fixed 45 degrees" offset from top-surface infill as the original design choice, the heat-creep warning, and the note that "the edges will be a tiny bit fuzzy or less sharp" due to nozzle size vs ironing path width. Explicit statement that "ironing is very sensitive to accurate extruder calibration — too little and shiny grooves will be visible, too much and excess plastic will be dragged by the nozzle to the edges". <https://blog.prusa3d.com/make-top-surfaces-super-smooth-ironing-prusaslicer-2-3-beta_41506/>

[^orca-wiki-ironing]: OrcaSlicer Wiki — Ironing Settings. Complete parameter reference: `ironing_type` (Top Surfaces / Topmost Surface / All solid layers), `ironing_pattern` (Concentric / Rectilinear), `ironing_flow` (percent of normal), `ironing_spacing` (recommended ≤ nozzle diameter), `ironing_inset` (distance from edges, 0 = at edge), `ironing_angle` (offset or absolute), `ironing_angle_fixed` (enables absolute angle for "uniform surface finish and reduced tiger striping effect"), `ironing_speed`. Includes the heat creep warning. <https://github.com/OrcaSlicer/OrcaSlicer/wiki/quality_settings_ironing>

[^bambu-ironing-types]: Bambu Lab community forum — Ironing type options differences explanation. Authoritative description of the three ironing types: `topmost` irons only the last layer of the model; `top` irons every top-facing surface; `all solid layers` irons all solid layers including internal solid infill and bottom layers, "rarely used because it does not necessarily produce good print quality but can take a lot of printing time". <https://forum.bambulab.com/t/ironing-type-options-differences-explenation/7055>

[^prusa-pr-cura]: Prusa PR #458 — "Implement Cura's Ironing Feature for a more smoother surface finish". The original feature request that led to PrusaSlicer 2.3 implementing ironing (which OrcaSlicer then inherited). Documents the Cura-style approach as the reference implementation. <https://github.com/prusa3d/PrusaSlicer/issues/458>

[^orca-discussion-1389]: OrcaSlicer Discussion #1389 — "Iron only top and bottom layer and/or choose ironing layers". Community request for per-layer ironing selection (paint-on). Not implemented; users currently work around with height-range modifiers. <https://github.com/SoftFever/OrcaSlicer/discussions/1389>

[^orca-calibration-issue]: OrcaSlicer Issue #5411 — "Add Calibration for Ironing". Community request for a built-in ironing calibration test generator. Closed as "not planned" in August 2024, though community interest persists. Users rely on MakerWorld/Printables calibration models instead. <https://github.com/SoftFever/OrcaSlicer/issues/5411>

[^orca-conditional-patterns]: OrcaSlicer Discussion #4997 — "Conditional Top Layer Patterns". Community request for the slicer to choose ironing pattern per-feature (concentric for round, rectilinear for rectangular). Not implemented. <https://github.com/SoftFever/OrcaSlicer/discussions/4997>

[^orca-issue-3796]: OrcaSlicer Issue #3796 — "Layer shift when using concentric top surface pattern". Documents the edge case where concentric ironing on small top surfaces produces a visible closing-point artefact. Workaround: use rectilinear for small top areas. <https://github.com/OrcaSlicer/OrcaSlicer/issues/3796>

[^orca-issue-angle]: OrcaSlicer Issue #10834 — discussion about ironing angle offset vs fixed. Recommends setting `ironing_angle_fixed: true` with angle 0° or 90° (perpendicular to a typical 45° top-surface infill direction) to "reduce tiger striping when reflecting light". <https://github.com/SoftFever/OrcaSlicer/issues/10834>

[^orca-issue-ironing-inset]: OrcaSlicer Issue #5812 — Ironing Inset. Documents the inset parameter and community recommendation to use 0.15-0.2 mm to prevent edge over-extrusion on perimeter-adjacent ironing paths. <https://github.com/SoftFever/OrcaSlicer/issues/5812>

[^orca-discussion-ironing-inset]: OrcaSlicer Discussion #2778 — "Feature Request: Ironing Inset Setting". Original community request for the inset parameter, with discussion of the nozzle-overshoot physics. Describes the failure mode where ironing all the way to the edge produces visible bulge or fuzz. <https://github.com/SoftFever/OrcaSlicer/discussions/2778>

[^all3dp-prusaslicer-ironing]: All3DP — "PrusaSlicer: Ironing – How to Get a Smooth Surface". Community-oriented guide with practical flow and speed recommendations: start at 10% flow for PLA, 15% for PETG/ABS, adjust in 2% increments. Notes that "ironing is sensitive to accurate extruder calibration — calibrating is a matter of trial and error". <https://all3dp.com/2/prusaslicer-ironing-simply-explained/>

[^snapmaker-ironing]: Snapmaker — "What is Ironing in 3D Printing? A Complete Guide". Detailed technical description including the requirement for 6 solid top layers at 0.2 mm height (1.2 mm of solid top thickness) to give the ironing pass something to work against. Also the rule of thumb that lower layer heights need proportionally higher ironing flow percentages. Explicit ABS warning about "sensitivity to temperature and airflow causing extrusion failures during the ironing process". <https://www.snapmaker.com/blog/ironing-in-3d-printing/>

[^prusa-knowledge-ironing]: Prusa Knowledge Base — Ironing heat creep section. Authoritative warning: "on some machines you might experience heat creep and eventually a clogged hotend because the extrusion is small and slow during ironing". PLA is "most prone to heat creep". Material-specific notes: PLA irons nicely but heat creep risk, PETG irons with nozzle-stick risk, ASA irons incredibly well, wood-filled filaments give poor results. <https://help.prusa3d.com/article/ironing_177488>

[^heat-creep-general]: Prusa Knowledge Base — Extrusion stopped mid-print (Heat creep). General heat creep reference: heat creep occurs when filament softens above the heat break due to slow filament movement or high ambient temp. PLA is particularly vulnerable due to its low glass transition (~60°C). Prevention: adequate hotend cooling, ambient temperature < 30°C, avoid very slow extrusion without necessity. <https://help.prusa3d.com/article/extrusion-stopped-mid-print-heat-creep_1948>

[^community-ironing-settings]: Community-tested ironing settings from Bambu Lab forums, Facebook 3D printing groups, and Reddit compilations: PLA 8-12% flow / 25-40 mm/s speed; PETG 15-35% flow (with 35% being surprisingly effective for some users) / 20-30 mm/s; ABS 12-18% flow / 30-40 mm/s; ASA 12-18% flow / 40-50 mm/s; TPU 25-30% flow / 15-20 mm/s (if ironed at all — most users disable). Spacing 0.08-0.15 mm depending on quality goals. <https://forum.bambulab.com/t/ironing-help/198091>

[^makerworld-ironing-test]: MakerWorld — "Ironing Test – Speed & Flow (10–25 mm/s | 5–20%)". Community calibration model with a grid that varies both speed and flow across a single print, letting users visually identify the optimum for their filament. Typical "sweet spot" identified by users is around 10-15% flow with 15-25 mm/s speed for PLA. <https://makerworld.com/en/models/1792469-ironing-test-speed-flow-10-25-mm-s-5-20>

[^makerworld-ironing-test-2]: MakerWorld — "Ironing calibration: speed&flow, 15-40mm/s & 10-50%". Larger range calibration model covering more extreme settings. Useful for finding the failure boundaries rather than the optimum. Community users typically land within the 10-25% flow / 20-40 mm/s speed box for most PLA variants. <https://makerworld.com/en/models/30075-ironing-calibration-speed-flow-15-40mm-s-10-50>

[^bambu-forum-small-areas]: Bambu Lab community forum — "Ironing top layer small areas with marks". Documented failure mode where ironing multiple small flat tops in a single print produces blob-at-restart and drag-mark artefacts. Workarounds include `topmost` ironing type and reducing ironing flow. <https://forum.bambulab.com/t/ironing-top-layer-small-areas-with-marks/98046>

[^bambu-forum-ironing-edges]: Bambu Lab community forum — "Ironing and edges". Documents the edge ghost-ring phenomenon where the ironing inset creates a visible transition between the un-ironed edge strip and the ironed interior. Fix options: reduce inset to 0 (accept fuzz) or increase to 0.3+ mm (make deliberate frame). <https://forum.bambulab.com/t/ironing-and-edges/244552>

[^prusa-forum-ironing-examples]: Prusa3D forum — "Okay, any examples of ironing IMPROVING prints?". Long community thread with before/after examples. Documents the gloss-mismatch phenomenon: ironed flat tops are glossier than non-ironed tilted tops on the same model, creating a visible finish discontinuity. Notes that matte filaments are worst affected. <https://forum.prusa3d.com/forum/prusaslicer/okay-any-examples-of-ironing-improving-prints/>
