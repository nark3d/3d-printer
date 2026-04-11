# Seam diagnosis — field notes from real prints

> **Source**: Adam's diagnostic findings from the past weeks of real-world print testing, collected via prior Claude chat sessions. **Hard-won experiential knowledge** — these are real diagnoses from real prints, with G-code-level root cause analysis. This document is the *empirical* counterpart to [`seams_deep_dive.md`](seams_deep_dive.md), which is the *theoretical* deep dive based on Adam L's guide and the OrcaSlicer wiki.
>
> **Read this when**: you're diagnosing a real seam problem on a real print. This document covers cases that have actually happened to *this* printer, not generic OrcaSlicer guidance.
>
> **Read [`seams_deep_dive.md`](seams_deep_dive.md) when**: you want to understand seams in general or look up an OrcaSlicer setting reference.
>
> **The two documents agree on the major findings** (scarf inner walls helps, wipe_on_loops hurts scarf, ERS at 300, 0.6 mm outer wall width, inner-outer-inner wall order). This document adds **new information** that isn't in Adam L's published guide — particularly G-code analysis techniques and specific failure mode signatures that only show up on certain print sizes or geometries.

---

## Why this document exists

Adam L's "Better Seams" Printables guide is the de facto community reference for scarf seams. It's exhaustive on the *settings* and *general principles*. But it's primarily based on testing on a Bambu X1C with PLA, and it doesn't cover certain failure modes that this printer has specifically encountered:

- **Pause / dead-stop at the scarf seam entry on large prints** (a G-code-level issue invisible from the slicer settings panel)
- **Seam gorges** caused by wipe settings being misconfigured in non-obvious ways
- **PA tuning interactions** with seam gap that can amplify pressure-advance errors
- **The scarf vs. normal seam wipe configuration tension** that has no clean resolution

These are documented here so we don't have to rediscover them.

---

## Case study 1 — Scarf seam pause on large ASA model

### Problem

A 1-2 second dead stop at the seam location on a large circular ASA print. Visible as a brief pause in the toolhead motion right before the scarf seam ramp begins on the outer wall. Only happened on large models, not on small test prints.

### Root cause (found via G-code analysis)

**The inner wall had a conventional hard-stop seam**, not a scarf seam. The G-code showed an `F120` (2 mm/s) deceleration command immediately before the outer wall scarf entry. That F120 was the inner wall preparing to dump pressure at its own traditional seam — a massive flow cliff that the toolhead had to physically slow down to handle.

The slicer panel didn't show this. Visual inspection of the OrcaSlicer settings was misleading. **You only saw it by reading the actual G-code output.**

### Why only on large models

Small models never reach sustained full speed before hitting the seam — the toolhead is still accelerating. So the flow delta at the seam is small and the deceleration is small. Large models cruise at full speed when they hit the seam, creating a **steep pressure cliff** that requires a large velocity drop to handle, hence the visible 1-2 second dead stop.

### Fix

**Enable scarf joint for inner walls** (`seam_slope_inner_walls: true`).

After enabling: the F120 deceleration disappeared from the G-code completely. The inner wall now ramps its flow gradually at the seam (just like the outer wall), so there's no pressure cliff and no need for the toolhead to decelerate.

### Lesson

> **Adam L's guide says scarf inner walls "often (but not always) improves results, but will slow down the print overall"** — the implication being it's a quality-vs-time trade-off. For *small prints* this is true. For *large prints* on this hardware, scarf inner walls is a **functional necessity** to prevent visible dead-stops at the seam. The slowdown from the inner wall scarf is more than offset by the elimination of the dead-stop pause.

This is **information that isn't in Adam L's published guide** — he tested on smaller geometries and didn't encounter this failure mode.

### Confirms

`seam_slope_inner_walls: true` (already in our config). **Override kept and now strongly justified by direct experience, not just Adam L's recommendation.**

---

## Case study 2 — Normal seam gorge (1 mm deep groove through full wall)

### Problem

Deep groove through the entire wall thickness on a cylinder. Visible on **both inner and outer surfaces** of the wall. The wall had effectively been cut through at the seam location for the full height of the print.

### Root cause (found via G-code analysis)

**Zero wipe moves in the entire G-code**, despite the wipe distance setting being configured to 2 mm in the slicer panel. The wipe distance setting in OrcaSlicer **does nothing on its own** — it requires `wipe_on_loops` or `wipe_while_retracting` to be enabled to actually generate wipe moves in the G-code.

The wall ends with the nozzle still under full pressure (no wipe = no pressure relief). The wall starts again with a pressure deficit after retraction (no de-retraction wipe = pressure builds slowly). Over many layers stacking the same defect at the same angle, the cumulative effect is a **gap through the entire wall thickness**.

### Compounding factor

Pressure advance was set too high (0.032). Per Klipper / Duet documentation: **"if there is a gap at the seam, PA is too high."** High PA over-retracts at the loop end and under-extrudes at the loop start, exaggerating the existing pressure dynamics.

### Hidden insight

`wipe_on_loops` only generates a tiny ~0.047 mm positional nudge — **not a real wipe at the configured 2 mm distance**. The "wipe distance" parameter only applies if there's a *real* wipe move to use it on. If neither `wipe_on_loops` nor `wipe_while_retracting` is enabled, no real wipes happen regardless of what you set the distance to.

This is non-obvious from the slicer UI because the wipe distance field is editable and has no warning that it's inert without enabling one of the other flags.

### Fix

1. **Reduced pressure advance from 0.032 to 0.028**
2. **Set seam gap to 10 %** (the default — was previously 0 %)
3. **Enabled `wipe_before_external_loop: true`** to generate real de-retraction wipes inside the model

After all three changes: gorge eliminated, normal seam visible but acceptable.

### Lesson

> **Seam gap at 0 % means the loop start and end meet *exactly*, amplifying any pressure dynamics issue. Default 10 % seam gap gives the slicer room to absorb pressure variation.** For traditional seams (not scarf), don't push seam gap to 0 — you'll catastrophically amplify any PA tuning error.

> **Wipe distance does nothing without `wipe_on_loops` or `wipe_while_retracting` enabled.** This is a UI footgun in OrcaSlicer.

### Conflicts

This case used `wipe_before_external_loop: true` — which **conflicts with Adam L's "things to avoid for scarf" list**. The reconciliation: this print was using *traditional* seams, not scarf. Wipe-before-external-loop helps traditional seams and hurts scarf seams. **The right setting depends on which seam mode you're using.**

---

## Case study 3 — Normal seam bump with ridge

### Problem

After fixing the gorge from Case 2, a small bump with a ridge appeared at the seam location. Smaller defect than the gorge, but still visible.

### Cause

Pressure advance was now too *low* (had been reduced to 0.026). Not pulling back enough material at the wall end → too much material deposited at the overlap point → bump.

### Fix

PA increased back to 0.028. Seam gap kept at 10 %.

### Lesson — PA tuning is a sweet spot, not "lower is better"

> Pressure advance has a narrow correct value for a given filament. **0.026 → bump (too low). 0.028 → clean. 0.032 → gorge (too high).** The window is roughly ±0.002 around the correct value. PA must be recalibrated per-filament because different filament formulations have different viscosity dynamics.

### Confirms

The Klipper documentation rule: **"gap at the seam → PA too high. Bump at the seam → PA too low."** Use this as a quick mental check when looking at any seam defect.

---

## Case study 4 — Scarf seam knotty mess at entry

### Problem

Messy blob at the scarf joint entry point. The scarf ramp itself was clean, but the *entry into the ramp* had a visible knot of extra material.

### Fix

**Changed `seam_slope_start_height` to 50 %** (default is 0).

### Result

Reduced to a small bump at scarf entry. Not eliminated entirely but significantly improved.

### Why this works

The default `seam_slope_start_height: 0` means the scarf ramp starts at the *current* layer Z and ramps up to full layer height over the scarf length. Setting it to 50 % means the ramp starts at *half* the layer height above the previous layer surface. The result is that the initial scarf segments don't contact the previous layer as aggressively, reducing the over-extrusion blob at the entry.

### Lesson

> **`seam_slope_start_height` is one of the few scarf settings that can be tuned to fix visible artefacts.** Adam L's guide doesn't focus on this setting (he recommends leaving it at default). Our experience says **try 50 % if the scarf entry shows a knotty blob**.

This is **new information not in Adam L's published guide**.

---

## Validation of Adam L's "Better Seams" guide rules

Tested on this hardware against real prints over the past weeks:

| Adam L's rule | Our experience |
|---|---|
| **Wipe on loops must stay OFF for scarf** | ✅ Confirmed — we've seen wipe_on_loops cause the artefacts Adam L describes |
| **Scarf flow ratio changes don't help** | ✅ Confirmed — tested down to 90 %, no improvement |
| **Scarf joint for inner walls should be ON** | ✅ **Strongly confirmed** — also fixes the F120 dead-stop pause on large prints (Case 1), which is a *bigger* benefit than the quality improvement Adam L describes |
| **Conditional scarf joint should be ON** | ✅ Confirmed |
| **Inner-outer-inner wall order recommended** | ✅ Confirmed |
| **ERS at 300 helps edge out quality** | ✅ Confirmed — *contradicts the Voron community's ~100 recommendation* but matches our experience |
| **Staggered inner seams recommended** | ✅ Confirmed — combats the inner nozzle landing mark near overhangs |
| **`retract_on_layer_change` ON + `wipe_while_retracting` ON together** | ✅ Confirmed — having both OFF causes problems |

**Note on ERS at 300**: This is *higher* than the Voron community's recommendation of ~100. Our experience matches Adam L's value (300), even though our hardware is more similar to a Voron than a Bambu X1C. **The Voron community's 100 mm³/s² value may be too conservative for our setup**. This contradicts the recommendation in [`speed_PLA.md`](speed_PLA.md) suggesting we lower ERS from 300 to 100-150. **The field experience says keep it at 300.**

---

## The fundamental wipe settings tension

### The problem

There's a real, unresolvable tension between scarf seams and traditional seams:

- **`wipe_on_loops` improves traditional seams** — it tucks the loop end inward to hide the seam point
- **`wipe_on_loops` hurts scarf seams** — it interferes with the scarf ramp and creates artefacts

If you have a print where *some* features use scarf and *some* use traditional (e.g. mixed curved and sharp features with `seam_slope_conditional` doing the per-feature decision), you can't optimise for both at once. You have to pick.

### How to decide

| If your print is... | Set `wipe_on_loops` to |
|---|---|
| Mostly curved surfaces (scarf will be active most of the time) | **`false`** |
| Mostly sharp corners (scarf rarely fires, traditional seams are dominant) | **`true`** |
| Mixed | **`false`** if visible quality matters more than functional strength on the corners |

### The 2024 backup profile worked around this

Adam's known-good 2024 profile **avoided the tension entirely** by running scarf on *everything* (`seam_slope_type: all`, no conditional disabling). It never printed a traditional seam at all, so the wipe_on_loops conflict never arose. **This is a defensible profile design choice** — accept that scarf will fire on every perimeter, including sharp corners where it's slightly suboptimal, in exchange for never having to deal with the wipe configuration conflict.

---

## The 2024 known-good profile

Documented for reference. This profile produced consistently good seams across many real prints in 2024 before the input shaper retuning, before the new shaper changes, and before the recent PA recalibration. Worth treating as a *baseline* to compare any changes against — if a new profile doesn't match the seam quality of this one, something has been lost.

| Setting | Value | Notes |
|---|---|---|
| **Seam position** | `nearest` (scattered, not aligned) | Per Adam L: random is incompatible with scarf, but `nearest` is compatible — and the scarf ramp handles seam visibility regardless of position |
| **Scarf joint type** | `all` (Contour and Hole) | Scarf on everything — sidesteps the wipe_on_loops tension |
| **Scarf inner walls** | enabled | Required for the F120 dead-stop fix on large prints |
| **ERS (`max_volumetric_extrusion_rate_slope`)** | **300** mm³/s² | Field-validated. Don't reduce to 100. |
| **Inner wall speed** | **320 mm/s** | Higher than my speed tab recommendations |
| **Outer wall line width** | **0.6 mm (150 %)** | Per Adam L's scarf-specific recommendation |
| **Outer wall speed** | **50 mm/s** | **Significantly slower than Adam L's 75-100 range and my speed tab recommendation of 100.** This is a *very* conservative outer wall speed. The reasoning: at 50 mm/s, scarf seams have maximum time to lay cleanly. Speed-vs-quality trade-off. |
| **Pressure advance** | 0.026 | Filament-dependent — this was the right value for the filament being used at the time |
| **Wall sequence** | `inner-outer-inner` | Required for scarf quality |

### Significant deviations from the current profile

| Setting | 2024 known-good | Current profile | Source of conflict |
|---|---|---|---|
| `outer_wall_speed` | **50** mm/s | 100 mm/s | The current value matches Adam L's *upper bound*, the 2024 value is much more conservative. The 2024 value worked. **Worth testing 50 mm/s for scarf-quality-critical prints.** |
| `outer_wall_line_width` | **0.6 mm** (150 %) | 0.4 mm (100 %, inherited) | This is the Adam L recommendation that I missed in the original Quality tab analysis. **The 2024 profile confirms it works in practice.** |
| `seam_slope_conditional` | likely off (scarf on everything) | `true` (conditional) | The 2024 approach was simpler — apply scarf everywhere, ignore the wipe_on_loops conflict |
| `inner_wall_speed` | **320 mm/s** | 180 mm/s | The 2024 profile pushed inner wall speeds higher than the current profile. The 2024 profile worked. |

### What this tells us

The 2024 profile was tuned by **prioritising visible seam quality above all else**. Outer wall speed was pushed *down* (50 mm/s) to give scarf seams maximum time to lay cleanly. Inner wall speed was pushed *up* (320 mm/s) to keep total print time reasonable. The outer wall width was *up* (0.6 mm) for better scarf flow stability. Scarf was applied to *everything* to avoid configuration conflicts.

The current profile is more *balanced* — better outer wall speeds, but at the cost of some scarf seam quality. **If we hit a print where scarf seams are critical, the 2024 profile is the known-good fallback.**

---

## Per-filament pressure advance values

Empirically tuned on this hardware:

| Filament | PA value | Notes |
|---|---|---|
| **eSUN PLA+ (green)** | **0.035** | |
| **Sunlu Matte PLA** | **0.045** | Higher than non-matte PLAs — matte additives change viscosity dynamics |

### Critical PA notes

- **PA is filament-specific** — different formulations need different values, even within "PLA"
- **PA must be recalibrated after changing input shaper settings** — input shaper changes the toolhead's effective acceleration profile, which interacts with PA timing
- The PA values above were valid *before* the recent input shaper retuning (LIS2DW MZV X=61 / Y=42). **They may need re-calibration** now that the shaper has changed
- Use OrcaSlicer's built-in PA calibration test print, not guesswork
- Klipper rule of thumb: "gap at the seam → PA too high; bump at the seam → PA too low"

---

## Key principles learned

These are general principles distilled from the case studies above. Worth memorising.

### 1. G-code analysis is essential

> **Visual inspection of the OrcaSlicer settings panel can be misleading about what actually happens in the toolpath.** The slicer UI shows you the *configured* values; the gcode shows you the *generated* moves. Sometimes the slicer doesn't generate what the UI suggests it would.

When diagnosing a seam problem:

1. Look at the gcode for the seam region directly
2. Search for unusual `F` (feedrate) values, especially low ones — they indicate the slicer is decelerating to handle a flow change
3. Search for `G1 E-` (retraction) and `G1 E+` (de-retraction) sequences around the seam — confirm the wipe and retraction moves are actually present
4. Compare the gcode against another print that *worked* — diff them if needed

Tools: `head` / `grep` / a text editor for the gcode file. Mainsail's gcode preview also visualises feedrate as colour gradients which can highlight deceleration points.

### 2. Seam gap of 0 amplifies pressure issues

> **Seam gap at 0 % means the loop start and end meet *exactly*, amplifying any pressure-advance error.** A 10 % default seam gap gives the slicer room to absorb pressure variation. **Don't push seam gap to 0 unless you have rock-solid PA calibration AND you specifically want zero overlap.**

For scarf seams: seam gap is automatically disabled on the external perimeter (per the original PR). For traditional seams: **keep seam gap at 10-15 %.**

### 3. On cylinders with N wall loops, the walls *are* the wall

> **There's no infill in a thin-walled cylinder. If your seam tunnels through the wall, it's tunnelling through the entire structural element of the print.** This is why the gorge in Case 2 was so dramatic — it wasn't a surface defect, it was a structural through-hole.

This also explains why Adam L found "single or double walls with no infill performed the very best" for thin curved features — there's less material stack to fight against scarf-induced flexing.

### 4. The outer→inner wall transition often skips retraction

> **At the outer-to-inner wall transition, OrcaSlicer often doesn't retract because the travel distance is under the minimum retraction threshold.** No retraction means no de-retraction means no wipe fires. This is per-segment behaviour that the slicer panel doesn't expose.

If you see seam-related artefacts at the *transition between perimeters* (not at the loop start/end), this is the likely cause. Either lower the minimum retraction travel distance, or accept that this transition won't have a wipe.

### 5. Scarf seams trade complexity for invisibility

> **Scarf seams are more sensitive to mis-tuning than traditional seams.** A traditional seam with bad PA shows a small bump or gap. A scarf seam with bad PA, wrong wipe settings, or wrong outer wall width shows a *bigger* artefact than the traditional seam would have. The pay-off is that when scarf is correctly tuned, it's *nearly invisible*. When traditional is correctly tuned, it's *visible but small*.

Scarf is the right choice when you have time to tune carefully and the print quality is worth the complexity. Traditional seams are the right choice when you're printing one-offs and don't want to fight the calibration.

---

## What this document adds beyond Adam L's published guide

Adam L's "Better Seams" guide is exhaustive on **settings and general principles**. This document adds **specific failure modes and diagnostic techniques** that Adam L didn't cover:

| Topic | Adam L's guide | This document |
|---|---|---|
| Wipe_on_loops conflict with scarf | Says "avoid" | Same recommendation, plus the exact mechanism (wipe distance is inert without the flag, the gorge it can cause when configured wrong) |
| Scarf inner walls | "Often improves results, slows print" | Same plus **the F120 dead-stop pause on large prints** that scarf inner walls explicitly fixes — a functional necessity, not just a quality improvement |
| Pressure advance | Mentioned as prerequisite | Specific Klipper rule ("gap = too high, bump = too low"), the seam gap × PA interaction, per-filament values |
| `seam_slope_start_height` | Recommends default | **50 % can fix the scarf entry blob** — empirical finding |
| ERS value | 300 on Bambu X1C | **300 also works on our Klipper setup** — contradicts the Voron 100 recommendation |
| Outer wall speed | 75-100 mm/s | Field-tested: **50 mm/s in the 2024 known-good profile** for absolute best scarf quality |
| G-code analysis | Not discussed | Dedicated section — what to look for and how to find it |

---

## Cross-references and pending updates to other documents

### Updates this document triggers in [`quality_PLA.md`](quality_PLA.md)

The quality tab document has two recommendations that should be revised based on field experience:

| Setting | Currently in `quality_PLA.md` | Should be |
|---|---|---|
| `outer_wall_line_width` | 105 % (general guidance) | **Note that scarf seam users should consider 150 % per Adam L + the 2024 known-good profile** |
| `wipe_on_loops` | `true` (default, per wiki) | **`false` when scarf seams are enabled** (per Adam L + Case 1 / Case 2 field experience) |

### Updates this document triggers in [`speed_PLA.md`](speed_PLA.md)

| Setting | Currently in `speed_PLA.md` | Should be |
|---|---|---|
| `max_volumetric_extrusion_rate_slope` | "Recommend reducing 300 → 100-150 (Voron community uses ~100)" | **Field experience says keep at 300** — Voron community's value may be too conservative for our specific setup. Defer to field experience. |
| `outer_wall_speed` | "Defensible at 100, could go to 120" | **Note that the 2024 known-good profile used 50 mm/s for scarf-critical prints** — significantly slower than my recommendation. The right value depends on whether scarf quality is the primary goal. |

### Updates this document triggers in [`seams_deep_dive.md`](seams_deep_dive.md)

The deep dive should reference this document at the top as the *empirical* counterpart to the *theoretical* deep dive. Both should be read together for full context.

---

## TL;DR for working sessions

**If you're tuning scarf seams from scratch on a new filament:**

1. Calibrate pressure advance for the filament *first* (don't skip this — it's the prerequisite)
2. Start from the 2024 known-good profile values
3. Print Adam L's test suite from Printables
4. If you see a dead-stop pause at the seam: enable scarf inner walls
5. If you see a knotty blob at scarf entry: try `seam_slope_start_height: 50 %`
6. If you see a gorge: check that you don't have `wipe_on_loops` and `wipe_while_retracting` *both* off, and check PA isn't too high
7. If you see a bump: PA is too low

**If you're getting a seam artefact you can't explain:**

1. Read the gcode at the seam region — look for low feedrate values and missing retraction/wipe sequences
2. Compare against a known-good print's gcode
3. If the gcode shows what you'd expect: it's probably PA or filament tuning
4. If the gcode shows something unexpected: it's probably a slicer setting interaction
