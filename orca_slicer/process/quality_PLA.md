# OrcaSlicer Quality tab settings — PLA quality profile

> One file per tab. See [`README.md`](README.md) for the index of all tab documents and shared hardware context.
>
> Working document. For each OrcaSlicer Quality tab setting: the OrcaSlicer default, the *researched* value (what reliable references — community profiles, wikis, vendor docs — recommend for this hardware combination), our current value, and a comments field documenting the reasoning behind our choice.
>
> **Hardware context**:
>
> - **Frame / kinematics**: ZeroG Mercury One.1 + Nebula Pro frame (CoreXY, 260 × 245 × 258 build volume) — *not* a VzBot 330, despite using a VzBot toolhead
> - **Toolhead**: VzBot CNC toolhead (aluminium variant, heavier than the printed Vz-Printhead the OrcaSlicer-bundled VzBot profile is calibrated for)
> - **Hotend**: Phaetus Rapido 2 HF (high-flow, claimed 45 mm³/s)
> - **Extruder**: Orbiter 2.5 (7.5 : 1 geared, ~0.06 mm measured backlash)
> - **Probe**: Beacon RevH (sub-micron Z probe σ measured)
> - **Motion limits**: `max_velocity` 600 mm/s, `max_accel` 10 000 mm/s², input shaper MZV X=61.0 Hz / Y=42.0 Hz (LIS2DW-derived)
> - **Material**: PLA, any brand
> - **Goal**: Quality-first
>
> **Why "VzBot toolhead, not VzBot printer" matters**: The OrcaSlicer-bundled `0.20mm Standard @Vzbot` process profile is calibrated for the *whole* VzBot 330 AWD with the *lightweight* Vz-Printhead (~22-27 g aluminium head). Our setup uses the heavier CNC toolhead variant on a Mercury One.1 frame. Direct consequences:
>
> | Category | Bundled VzBot profile is… |
> |---|---|
> | Hotend flow / volumetric speed | Directly applicable (same hotend) |
> | Extruder retraction / pressure advance | Directly applicable (same extruder) |
> | Quality (line widths, walls, seams, ironing, bridging flow) | Mostly applicable (toolhead-mass-independent) |
> | Speeds | An *upper bound* — heavier toolhead → slower viable speeds |
> | Accelerations | An *upper bound* — lower-frequency input shaper means less accel headroom |
> | Frame / kinematics options | Not applicable — different printer |
>
> **Inheritance reminder**: OrcaSlicer process profiles are inheritance-based. Our PLA profile resolves through `0.20mm Standard @MyKlipper - PLA` → `0.20mm Standard @MyKlipper` → `fdm_process_klipper_common` → `fdm_process_common`. The "OrcaSlicer default" column is the value we arrive at *before* any of our overrides, walking that chain from `fdm_process_common`.
>
> **Column meanings**:
>
> - **Setting**: UI label as shown in OrcaSlicer
> - **OrcaSlicer default**: Value Orca arrives at without overrides. Where I couldn't confirm with confidence I've marked it explicitly (`?` or `(probably X)`) rather than fabricate a number
> - **Researched value**: What community references / vendor docs / curated profiles recommend, with footnote citations to the source
> - **Our value**: What our `0.20mm Standard @MyKlipper - PLA` profile currently has set. `(default)` means we don't override and inherit the OrcaSlicer default
> - **Comments**: Reasoning for our choice — especially when we differ from either the default or the researched value
>
> **Confidence note**: Where the "Researched value" column says "no specific research found", that means I couldn't find an authoritative reference and the value in the column is OrcaSlicer's default (which is generally a sensible starting point for most builds).

---

## Quality tab

### Layer height

| Setting | OrcaSlicer default | Researched value | Our value | Comments |
|---|---|---|---|---|
| Layer height | 0.2 mm | **0.2 mm** [^layer-rule] | 0.2 mm *(inherited)* | 50 % of nozzle diameter. Universal "Standard" choice for quality-first PLA. **DECISION: no change** — 0.2 mm is correct for this profile. |
| First layer height | 0.2 mm | 0.2-0.25 mm [^first-layer] | 0.2 mm *(inherited)* | With the Beacon Z probe (σ ≈ 60 nm), we don't need extra adhesion margin. **DECISION: no change** — default 0.2 mm fine; only bump if first-layer issues appear. |

### Line width

| Setting | OrcaSlicer default | Researched value | Our value | Comments |
|---|---|---|---|---|
| Default | 110 % (0.44 mm) | 0 (auto) [^line-width] | **0** | Override to 0 = "auto, defer to per-feature widths". Cleanest approach — per-feature widths below override this. **DECISION: no change** — override is correct. |
| First layer | 120 % (0.48 mm) | 120 % (or 0.5 mm) [^vzbot-bundled] [^line-width-bambu] | 120 % *(inherited)* | Wider first layer = better bed adhesion. Within the 105-150% range. **DECISION: no change** — default 120% is correct. |
| Outer wall | 100 % (0.4 mm) | **105 %** (0.42 mm) [^vzbot-bundled] [^line-width-bambu] | 100 % *(inherited)* | Community consensus: 105-120% for outer wall. VzBot bundled uses 105%. Orca wiki's 100% is the conservative position. **DECISION: CHANGE to `105 %`** — slightly better detail and inter-wall bonding, no meaningful dimensional accuracy loss (0.02 mm is below XY backlash anyway). |
| Inner wall | 110 % | **112-120 %** [^vzbot-bundled] [^line-width-bambu] | 110 % *(inherited)* | Community recommends ≥ 120% for max inter-wall bonding. Our 110% is conservative. **DECISION: CHANGE to `115 %`** — middle of the community range, better wall bonding without affecting dimensions. |
| Top surface | 93.75 % (0.375 mm) | 100-105 % [^line-width-bambu] | **100 %** | Override to 100% is correct — Bambu/community recommends ≥100% for top surface (the default 93.75% is unusually conservative). **DECISION: no change** — override is correct. |
| Sparse infill | 110 % | 110-150 % [^line-width-bambu] | **120 %** | Override to 120% is in the quality-focused end of the range. Could go higher for speed but we're quality-first. **DECISION: no change** — 120% is a good quality choice. |
| Internal solid infill | 120 % | 105-120 % [^vzbot-bundled] [^line-width-bambu] | 120 % *(inherited)* | Either 105% or 120% works. 120% closes gaps slightly better. **DECISION: no change** — default is correct. |
| Support | 96 % (0.384 mm) | 96-100 % [^vzbot-bundled] | 96 % *(inherited)* | Narrower support lines are easier to peel away. **DECISION: no change** — default is the common-preference choice. |

### Seam

| Setting | OrcaSlicer default | Researched value | Our value | Comments |
|---|---|---|---|---|
| Seam position | `aligned` | **`aligned_back`** [^seam-wiki] [^seam-position] | **`nearest`** | Community consensus per multiple sources: **"For most prints, Aligned or Back gives the cleanest result."** `Random` is only for organic shapes. `Nearest` (our current value) minimises travel time but spreads visible seam scars across all sides of the print. `Aligned_back` puts the seam consistently on the rear face where it's not visible to the viewer. **Strong recommend changing** to `aligned_back`. With our scarf seam already enabled, the seam itself becomes nearly invisible regardless of position, but `aligned_back` ensures any residual visibility is on the hidden face. |
| Staggered inner seams | `false` | `true` [^seam-wiki] | **`true`** | Offsets internal-perimeter seams from the layer above so they don't form a vertical column of weak spots. Quality benefit, no real cost. |
| Seam gap | 10 % | 0-15 % [^seam-wiki] (VzBot uses 2 % [^vzbot-bundled]) | **5 %** | We're tighter than the 10 % default. VzBot bundled uses 2 % (even tighter). 5-15 % is the optimal range per the wiki. With scarf seam enabled the gap matters less. |
| Scarf joint seam | `none` | `all` (Contour and hole) [^seam-wiki] | **`all`** | The flagship visible-quality feature in OrcaSlicer. Ramps flow at seam start/end so there's no visible seam scar. Must-have for PLA quality. |
| Conditional scarf joint | `false` | `true` [^seam-wiki] | **`true`** | Only applies the scarf ramp on curved perimeters where it helps. Skips sharp corners where scarf doesn't help and would slow things down. |
| Conditional angle threshold | 155° | 155° [^seam-wiki] | 155° *(inherited)* | Angle below which a corner is considered "sharp" and scarf is skipped. Wiki default is well-tuned. |
| Conditional overhang threshold | 40 % | 40 % [^seam-wiki] | 40 % *(inherited)* | Skip scarf where the perimeter has more than 40 % overhang above. Prevents scarf seams from sagging. |
| Scarf joint speed | 100 % | < 100 mm/s absolute [^seam-wiki] | 100 % *(inherited)* | Speed multiplier for scarf transitions. Wiki recommends scarf at < 100 mm/s absolute. Our outer wall is 100 mm/s, so 100 % multiplier puts us right at the recommended ceiling. Reduce to 70-80 % if scarf artefacts appear at high outer-wall speeds. |
| Scarf joint length | ~20 mm | 20 mm [^seam-wiki] [^scarf-guide] | 20 mm *(inherited)* | Per the comprehensive community scarf seam guide on Printables (tested against PLA), the default 20 mm "generally offers smooth and effective blending". A value of 0 disables the scarf joint. Higher values for severe-seam cases. |
| Scarf joint steps | 10 | 10-20 [^seam-wiki] [^scarf-guide] | **20** | Community guide states: "More steps create a smoother gradient, but the default value (10) is sufficient in most cases." We override to 20 — smoother flow ramp at the cost of more gcode lines. With our 600 mm/s motion system the gcode count isn't a limit. Defensible quality choice; the default 10 would also be fine. |
| Scarf joint flow ratio | 100 % | 100 % [^seam-wiki] | 100 % *(inherited)* | Don't change unless scarf-specific flow artefacts show up. |
| Scarf for inner walls | `false` | (no community consensus) | **`true`** | We override to `true`. Inner-wall seams are normally hidden but can show through on thin walls or as ghosting. Slight time cost. Defensible quality choice. |
| Around entire wall | `false` | `false` [^seam-wiki] | `false` *(inherited)* | If `true`, scarf wraps the entire perimeter loop. Huge time cost for marginal quality gain. Leave default. |
| Role-based wipe speed | `true` | `true` [^seam-wiki] | `true` *(inherited)* | Nozzle wipe at the end of a feature happens at the same speed as the feature. Sensible. |
| Wipe speed | 80 % | 80 % [^seam-wiki] | 80 % *(inherited)* | Only used if role-based wipe is off. N/A. |
| Wipe on loop | `true` | `true` [^seam-wiki] | `true` *(inherited)* | Tucks the loop end inward to hide the seam point. Works in concert with scarf seam. |
| Wipe before external loop | `false` | `false` [^seam-wiki] | `false` *(inherited)* | Adds a de-retract move inside the model before each external loop. Marginal benefit, can cause artefacts on small features. |

### Precision

| Setting | OrcaSlicer default | Researched value | Our value | Comments |
|---|---|---|---|---|
| Slice gap closing radius | 0.049 mm | 0.049 mm [^precision-wiki] | 0.049 mm *(inherited)* | Mesh-level repair tolerance for STL gaps. **DECISION: no change** — default is correct unless we have specifically buggy STLs. |
| Resolution | 0.012 mm | 0.012 mm [^precision-wiki] | 0.012 mm *(inherited)* | Contour simplification tolerance. Lower values bloat gcode for no visible benefit. **DECISION: no change** — default is correct. |
| Arc fitting | `false` | **`false`** for Klipper [^arc-fitting-klipper] | **`false`** *(explicit)* | **CORRECTED 2026-04-11**: earlier research claimed `true` based on VzBot bundled profile. That was wrong for Klipper. Two reasons: (1) **ERS mutual exclusion** — OrcaSlicer's UI disables arc fitting whenever `max_volumetric_extrusion_rate_slope > 0`. We run ERS at 300 (Adam L's value, validated in field notes). (2) **Klipper converts G2/G3 back to line segments internally** via `[gcode_arcs] resolution`, so enabling arc fitting creates a double lossy conversion: slicer G1→G2/G3 (lossy), then Klipper G2/G3→G1 (lossy). Per OrcaSlicer Issue #4216 devs: "Arc fitting is discouraged for Klipper-based machines". **Note on ZeroG docs**: ZeroG recommends enabling `[gcode_arcs]` in Klipper printer.cfg — that's a **separate** thing from slicer-side arc fitting. `[gcode_arcs]` just lets Klipper *parse* G2/G3 if they appear; it doesn't recommend the slicer *produce* them. We have `[gcode_arcs]` enabled in printer.cfg (correct), arc fitting disabled in slicer (correct). **DECISION: SET EXPLICITLY to `0`** — don't inherit false, lock it in. |
| X-Y hole compensation | 0 mm | **0.075 mm** [^vzbot-bundled] [^xy-hole] | 0 *(inherited)* | VzBot bundled value. **Caveat**: can cause ridge artefacts on holes with partial openings to the outside (GitHub Issue #8011). **DECISION: CHANGE to `0.075 mm`** — the artefact case is rare enough for most prints. If you hit it, disable per-print. |
| X-Y contour compensation | 0 mm | 0 mm [^vzbot-bundled] | 0 *(inherited)* | Only needed for specific shrinking filaments. **DECISION: no change** — default is correct. |
| Elephant foot compensation | 0.15 mm | **0.1 mm** [^vzbot-bundled] [^elephant-foot] | 0.15 mm *(inherited)* | Tuned printers with accurate first-layer Z (Beacon σ≈60 nm) need *less* compensation, not more. VzBot bundled uses 0.1 mm. **DECISION: CHANGE to `0.1 mm`**. |
| Elephant foot compensation layers | 1 | 1 *(no community guidance)* | 1 *(inherited)* | Number of layers over which compensation is graduated. **DECISION: no change** — default is correct. |
| Precise wall | `false` | **`true`** (with `inner-outer-inner` wall order) [^precise-wall] | `false` *(inherited)* | Sets outer-to-inner overlap to zero → prevents inner wall pushing out the outer. Explicitly recommended with our `inner-outer-inner` wall order. Klipper supports G2/G3 arcs fine. **DECISION: CHANGE to `true`**. |
| Precise Z height | `false` | `false` *(no community guidance)* | `false` *(inherited)* | Adjusts last 5 layers to exactly match model height. Costs slicing time, can cause visible Z banding. **DECISION: no change** — leave off unless specifically needed for a mating part. |
| Polyhole conversion | `false` | `false` *(no specific research)* | `false` *(inherited)* | Converts circular holes to faceted polygons. Causes visible faceting — XY hole compensation is the better approach. **DECISION: no change** — leave off. |
| Polyhole threshold | 0.01 mm | — | — *(inherited)* | Only matters if polyhole is on. **DECISION: no change** — N/A. |
| Polyhole twisted | `false` | — | — *(inherited)* | Only matters if polyhole is on. **DECISION: no change** — N/A. |

### Ironing

| Setting | OrcaSlicer default | Researched value | Our value | Comments |
|---|---|---|---|---|
| Ironing type | `no ironing` | `top surfaces` | **`top surfaces`** *(implied — flow set, type not explicitly shown)* | Ironing flattens visible top surfaces by re-running the nozzle over them at low flow. Single biggest top-surface quality improvement for PLA. `topmost surface` is even more selective and saves time on tall prints with multiple top surfaces. |
| Ironing pattern | `Concentric` | `Concentric` [^ironing-wiki] | `Concentric` *(inherited)* | Concentric is the standard for PLA. Rectilinear is faster but slightly more visible. |
| Ironing flow | 10 % | 10-15 % [^ironing-wiki] | **12 %** | Slightly more flow than default to ensure the nozzle drag deposits a tiny amount of fresh material to fill voids. Within the recommended range. |
| Line spacing | 0.15 mm | 0.1-0.15 mm [^ironing-wiki] | **0.1 mm** | Tighter spacing = smoother result but slower. 0.1 mm is on the quality end. |
| Inset | (probably 0.21 mm) | default [^ironing-wiki] | *(inherited)* | Distance to keep from the edges to prevent over-extrusion. Default fine. |
| Angle offset | -1 (auto) | **0° or 90°** (when solid infill is at default 45°) [^ironing-angle] | **90°** | OrcaSlicer Issue #10834 and the wiki recommend using a fixed angle that's offset from the solid infill direction (default 45°). 0° or 90° creates a uniform direction that "reduces tiger striping when reflecting light". Our 90° matches the recommendation. |
| Fixed angle | `false` | **`true`** [^ironing-angle] | **`true`** | Enables the fixed angle above instead of the auto-rotating default. |

### Walls and surfaces

| Setting | OrcaSlicer default | Researched value | Our value | Comments |
|---|---|---|---|---|
| Wall sequence | `inner wall/outer wall` | `inner-outer-inner` [^wall-order] | **`inner-outer-inner wall`** | Best external surface finish and dimensional accuracy. Requires ≥3 walls (we have 4). Slight overhang penalty accepted for quality goal. **DECISION: no change** — override is correct. |
| Detect overhang walls | `true` | `true` [^vzbot-bundled] | `true` *(inherited)* | Lets per-overhang-percentage speed slowdowns take effect. Quality necessity. **DECISION: no change** — default is correct. |
| Detect thin walls | `false` | `false` [^vzbot-bundled] | `false` *(inherited)* | Causes artefacts more often than it helps — Arachne handles thin features better. **DECISION: no change** — leave off. |
| Single wall on top surfaces | `false` | `true` (commonly enabled for quality) | `false` *(inherited)* | One perimeter on top layers → more room for top-surface infill that ironing flattens. Widely-applied community quality practice. **DECISION: CHANGE to `true`**. |
| Threshold for one wall on top | `min_width_top_surface` | (default) [^wall-and-surfaces-wiki] | *(inherited)* | Only applies single-wall-top when surface is this wide. Default prevents triggering on narrow features. **DECISION: no change** — default is correct. |
| Single wall first layer | `false` | `false` *(no community consensus)* | `false` *(inherited)* | First layer benefits from full perimeters for adhesion. **DECISION: no change** — off is correct. |
| Ensure vertical shell thickness | `enabled` | **`moderate`** (caution against `all`) [^vertical-shell] | `enabled` *(inherited)* | Multiple GitHub issues report `all` causes a visible "hull line" artefact on Benchies/curved surfaces. `moderate` prevents worst gaps without the artefact. The legacy `enabled` may map to `all`. **DECISION: CHANGE explicitly to `moderate`**. |
| No bridges over no-support areas | `false` | `false` [^vzbot-bundled] | `false` *(inherited)* | Bridges are useful — leave off. **DECISION: no change**. |
| Thick bridges | `false` | `false` [^vzbot-bundled] | `false` *(inherited)* | Thin bridges sag less. **DECISION: no change** — leave off. |
| Counterbore hole bridging | `partially bridged` | (no specific research) | *(inherited)* | Edge case for counterbored holes. **DECISION: no change** — default is fine. |
| Reduce crossing wall | `false` | `false` [^vzbot-bundled] | **`true`** | Override routes infill/travel moves to avoid crossing the visible outer wall. Prevents stringing scars on visible surfaces. **DECISION: no change** — override is correct for quality goal. |
| Max travel detour distance | 0 (unlimited) | 0 [^vzbot-bundled] | 0 *(inherited)* | Maximum extra travel to avoid crossing walls. 0 = always detour regardless of distance. Fine on a 260 mm bed. **DECISION: no change** — default is correct. |

### Wall generator

| Setting | OrcaSlicer default | Researched value | Our value | Comments |
|---|---|---|---|---|
| Wall generator | `classic` | `arachne` (modern recommended) | `classic` *(inherited)* | Arachne is the modern variable-width wall algorithm — handles thin features (text, fine details, narrow gaps) far better than classic. Slight slicing time cost, real quality benefit. **DECISION: CHANGE to `arachne`**. |
| Wall transition angle | 10° | 10° [^wall-gen-wiki] | 10° *(inherited)* | Arachne-only. Minimum angle for adding a transition between odd/even wall counts. **DECISION: no change** — default is correct. |
| Wall transition filter margin | 25 % | 25 % [^wall-gen-wiki] | 25 % *(inherited)* | Arachne-only. Hysteresis to prevent rapid wall-count switching. **DECISION: no change** — default is correct. |
| Wall transition length | 100 % | 100 % [^wall-gen-wiki] | 100 % *(inherited)* | Arachne-only. Wall-count transition distance. **DECISION: no change** — default is correct. |
| Wall distribution count | 1 | 1 [^wall-gen-wiki] | 1 *(inherited)* | Arachne-only. Default 1 = only the innermost wall varies width, protecting visible outer walls. **DECISION: no change** — default is correct. |
| Minimum wall width | 85 % | 85 % [^wall-gen-wiki] | 85 % *(inherited)* | Arachne-only. Narrowest printable wall. **DECISION: no change** — default is correct. |
| First layer minimum wall width | 85 % | 85 % [^wall-gen-wiki] | 85 % *(inherited)* | Same but for first layer. **DECISION: no change** — default is correct. |
| Minimum feature size | 25 % | 25 % [^wall-gen-wiki] | 25 % *(inherited)* | Smallest feature width that gets printed. **DECISION: no change** — default is correct. |
| Minimum wall length | 0.5 | 0.5 *(no community guidance)* | 0.5 *(inherited)* | Avoids extremely short isolated wall segments. **DECISION: no change** — default is correct. |

### Bridging (Quality tab portion)

| Setting | OrcaSlicer default | Researched value | Our value | Comments |
|---|---|---|---|---|
| Bridge flow ratio | 1.0 | **0.95** [^vzbot-bundled] [^bridge-flow] | 1.0 *(inherited)* | VzBot bundled ships 0.95. Community standard for reducing PLA bridge sag. **DECISION: CHANGE to `0.95`**. |
| Internal bridge flow ratio | 1.5 | **1.0** (or even 0.9) [^internal-bridge] | 1.5 *(inherited)* | Default 1.5 over-flows and multiple GitHub issues report this causes lumps that protrude above the current layer, sometimes causing nozzle crashes. 1.0 is modern community consensus. **DECISION: CHANGE to `1.0`**. |
| Top surface flow ratio | 1.0 | 1.0 *(no community guidance)* | 1.0 *(inherited)* | Filament flow ratio is the better lever. **DECISION: no change** — default is correct. |
| Initial layer flow ratio | 1.0 | 1.0 *(no community guidance)* | 1.0 *(inherited)* | Filament flow ratio is the better lever. **DECISION: no change** — default is correct. |

### Overhang and slowdown (Quality tab portion)

| Setting | OrcaSlicer default | Researched value | Our value | Comments |
|---|---|---|---|---|
| Slow down for overhang | `true` | `true` *(no specific research)* | `true` *(inherited)* | Quality necessity — off would print all overhangs at full speed. **DECISION: no change** — default is correct. |
| Slow down for curled perimeters | `true` | `true` *(no specific research)* | `true` *(inherited)* | Detects when a previous layer's overhang has curled and slows to avoid collision. **DECISION: no change** — default is correct. |
| Number of slow layers | 0 | 2-5 [^slow-down-layers] | **2** | Override ramps speed up over 2 layers after the initial layer. Community range is 2-5; our 2 is conservative end. **DECISION: no change** — override is defensible. |
| Wipe nozzle on top surfaces | `false` | `false` *(no specific research)* | `false` *(inherited)* | Useful for stringy filaments only. **DECISION: no change** — leave off. |

### Other quality

| Setting | OrcaSlicer default | Researched value | Our value | Comments |
|---|---|---|---|---|
| Adaptive layer height | `false` | `false` [^vzbot-bundled] [^adaptive-layer] | `false` *(inherited)* | Layer-height transitions create visible banding on walls. VzBot bundled keeps it off. **DECISION: no change** — leave off for PLA quality goal. |

---

## Summary of recommended Quality tab changes from current overrides

The Quality tab analysis identifies these settings where the **researched value** differs from our **current value**. Sorted by confidence (well-supported recommendations first, optional refinements last):

### Strongly recommended (multiple sources agree, clear quality benefit, no real downside)

| Setting | Currently | Recommend | Reason |
|---|---|---|---|
| `seam_position` | `nearest` | **`aligned_back`** | Multiple community sources: "For most prints, Aligned or Back gives the cleanest result." Quality-best — invisible seams on rear face. With our scarf seam, even more important. |
| `enable_arc_fitting` | `false` *(inherited)* | **`false`** (CORRECTED) | **Original recommendation was wrong.** ERS at 300 mm³/s² mutually excludes arc fitting (OrcaSlicer UI forcibly disables it). Also, arc fitting on Klipper produces double lossy conversion (slicer G1→arc, Klipper arc→G1). Per OrcaSlicer Issue #4216: "discouraged for Klipper". See corrected note in the Precision section above. |
| `bridge_flow` | `1.0` *(inherited)* | **`0.95`** | VzBot bundled value; community standard for reducing PLA bridge sag. |
| `internal_bridge_flow` | `1.5` *(inherited)* | **`1.0`** | Multiple GitHub issues report the default 1.5 over-flow causes lumps that protrude above the current layer (sometimes nozzle crashes). 1.0 is the modern community consensus. |
| `wall_generator` | `classic` *(inherited)* | **`arachne`** | Modern variable-width algorithm, better thin-feature handling. Slight slicing time cost, real quality benefit. |
| `precise_outer_wall` | `false` *(inherited)* | **`true`** | Small dimensional accuracy gain, recommended specifically with our `inner-outer-inner` wall order. No real downside. |
| `elefant_foot_compensation` | `0.15 mm` *(inherited)* | **`0.1 mm`** | Community default and VzBot bundled value. Tuned printers with accurate first-layer Z (which we have via Beacon) typically need *less* compensation. |
| `ensure_vertical_shell_thickness` | `enabled` *(inherited)* | **`moderate`** | The `all` value (which `enabled` may map to) is reported in multiple GitHub issues to cause a visible "hull line" artefact. `moderate` is the safer middle ground. |

### Recommended (community consensus, clear small improvement)

| Setting | Currently | Recommend | Reason |
|---|---|---|---|
| `outer_wall_line_width` | `100 %` *(inherited)* | **`105 %`** (0.42 mm) | Bambu/community recommends 105-120 % for outer wall. VzBot bundled uses 105 %. The OrcaSlicer wiki's 100 % advice is more conservative than community practice. Slightly better detail and inter-wall bonding. |
| `xy_hole_compensation` | `0` *(inherited)* | **`0.075 mm`** | VzBot bundled value. **Caveat**: causes visible ridge artefacts on holes that have a *partial opening to the outside* (per GitHub Issue #8011). If your prints typically have such features, leave at 0. |
| `only_one_wall_top` | `false` *(inherited)* | **`true`** | Common community quality practice. More room for top-surface ironing to flatten. |

### Optional refinements (small improvements, not blocking)

| Setting | Currently | Could refine to | Reason |
|---|---|---|---|
| `inner_wall_line_width` | `110 %` *(inherited)* | 112-115 % | Community recommends ≥ 120 % for max inter-wall bonding. We're slightly conservative. |

### Confirmed correct (no change needed)

The following settings we already have set well and the research confirms our choices:

- `wall_sequence: inner-outer-inner wall` — best surface quality (with caveat: slightly worse on overhangs)
- `seam_slope_type: all` (scarf seam) — flagship quality feature
- `staggered_inner_seams: true`
- `seam_slope_conditional: true`
- `seam_slope_inner_walls: true`
- `seam_slope_steps: 20`
- `seam_gap: 5%`
- `ironing_type: top surfaces`
- `ironing_flow: 12%`
- `ironing_spacing: 0.1 mm`
- `ironing_angle: 90°` with `ironing_angle_fixed: true`
- `reduce_crossing_wall: true`
- `top_surface_line_width: 100 %`
- `slow_down_layers: 2`

None of the recommended changes are urgent. None will fix a current problem — they're "would slightly improve quality on the next print". Apply selectively based on what you actually print.

---

## References

[^layer-rule]: OrcaSlicer Wiki — Layer Height. The 20-80 % of nozzle diameter rule. <https://github.com/SoftFever/OrcaSlicer/wiki/quality_settings_layer_height>

[^first-layer]: Same source. First layer height of 0.25 mm recommended for 0.4 mm nozzle in the wiki, though 0.2 mm is also commonly used and is the OrcaSlicer default. <https://github.com/SoftFever/OrcaSlicer/wiki/quality_settings_layer_height>

[^line-width]: OrcaSlicer Wiki — Line Width. Recommends 100 % of nozzle for outer wall (dimensional accuracy), 105-150 % for other features. <https://github.com/SoftFever/OrcaSlicer/wiki/quality_settings_line_width>

[^vzbot-bundled]: OrcaSlicer-bundled VzBot process profile (`fdm_process_Vzbot_common.json`). Maintained by VzBot devs and shipped with OrcaSlicer. **Caveat**: calibrated for the *lightweight* Vz-Printhead on the VzBot 330 AWD frame. Speed/acceleration values are an upper bound for our heavier CNC toolhead variant. Quality-related values (line widths, bridging flow, hole compensation) transfer fine since they're toolhead-mass-independent. **Note**: VzBot profile has `enable_arc_fitting: true` but this does NOT transfer to our Klipper setup — see `[^arc-fitting-klipper]`. <https://github.com/SoftFever/OrcaSlicer/tree/main/resources/profiles/Vzbot/process>

[^arc-fitting-klipper]: **Arc fitting is not recommended on Klipper** and is mutually exclusive with ERS (Extrusion Rate Smoothing) in the OrcaSlicer UI. (1) **ERS mutex**: OrcaSlicer disables the `enable_arc_fitting` checkbox whenever `max_volumetric_extrusion_rate_slope > 0`. Our profile runs ERS at 300 mm³/s² (Adam L's value, validated in our field notes). Source: [OrcaSlicer Issue #4216](https://github.com/SoftFever/OrcaSlicer/issues/4216) — quote from project collaborator: "ERS does not play ball with Arc fitting. so it is disabled." (2) **Double lossy conversion on Klipper**: even if ERS were off, arc fitting on Klipper creates two lossy conversions: slicer converts G1→G2/G3 (geometric approximation), then Klipper converts G2/G3→line segments via `[gcode_arcs] resolution: 1mm` default (another approximation). Community consensus ([Creality forum — Arc Fitting in process profiles for Klipper based machines](https://forum.creality.com/t/arc-fitting-in-process-profiles-for-klipper-based-machines/26232)): leave arc fitting OFF in slicer, keep `[gcode_arcs]` ON in printer.cfg. The two settings are NOT the same thing — `[gcode_arcs]` lets Klipper *parse* arc commands if they appear; `enable_arc_fitting` makes the slicer *produce* them. ZeroG recommends the former (we have it), not the latter. PR #5352 updated OrcaSlicer docs to make this explicit.

[^seam-wiki]: OrcaSlicer Wiki — Seam Settings. Authoritative source for scarf joint, seam position, seam gap, and wipe options. <https://github.com/SoftFever/OrcaSlicer/wiki/quality_settings_seam>

[^precision-wiki]: OrcaSlicer Wiki — Precision Settings. Slice gap closing, resolution, arc fitting, XY compensation, elephant foot, precise wall/Z. <https://github.com/SoftFever/OrcaSlicer/wiki/quality_settings_precision>

[^ironing-wiki]: OrcaSlicer Wiki — Ironing Settings. Type, pattern, flow %, spacing, inset, angle. <https://github.com/SoftFever/OrcaSlicer/wiki/quality_settings_ironing>

[^wall-gen-wiki]: OrcaSlicer Wiki — Wall Generator (Arachne). Wall transition angle, filter margin, length, distribution count, minimum wall width, minimum feature size. <https://github.com/SoftFever/OrcaSlicer/wiki/quality_settings_wall_generator>

[^rapido2-flow]: Phaetus Rapido 2 HF maximum volumetric flow specifications. Manufacturer claim: 45 mm³/s. Andrew Ellis Print Tuning Guide for Rapido HF (v1): ~24 mm³/s. Voron community real-world testing for Rapido HF: 15-20 mm³/s default, 20+ mm³/s with extruder current optimisation. Practical PLA target on Rapido 2 HF: 22-28 mm³/s. *Used in the Speed tab and filament profile, not in the Quality tab — listed here for the upcoming Speed tab section.* <https://github.com/AndrewEllis93/Print-Tuning-Guide/blob/main/articles/determining_max_volumetric_flow_rate.md> · <https://forum.vorondesign.com/threads/maximum-volumetric-flow-rate.195/>

[^orbiter25]: Orbiter 2.5 extruder specifications. 7.5 : 1 gear ratio, ~0.06 mm measured backlash. Manufacturer recommended retraction: 1.2 mm. Voron / RatRig community range for PLA: 0.4-0.8 mm. *Used in the filament profile, not in the Quality tab — listed here for reference.* <https://www.orbiterprojects.com/orbiter-v2-5/>

[^wall-order]: OrcaSlicer Wall and Surfaces Wiki + community discussion confirming inner-outer-inner gives best surface quality and dimensional accuracy, with worse overhang performance as the trade-off, and requires minimum 3 walls to be effective. <https://github.com/SoftFever/OrcaSlicer/wiki/quality_settings_wall_and_surfaces> · GitHub Issue #6789 · <https://github.com/SoftFever/OrcaSlicer/discussions/4697>

[^wall-and-surfaces-wiki]: OrcaSlicer Wall and Surfaces Wiki — documents `wall_sequence`, `is_infill_first`, `wall_direction`, `only_one_wall_top`, `only_one_wall_first_layer`, `min_width_top_surface`, `reduce_crossing_wall`, `max_travel_detour_distance`, `small_area_infill_flow_compensation`. <https://github.com/SoftFever/OrcaSlicer/wiki/quality_settings_wall_and_surfaces>

[^vertical-shell]: Multiple OrcaSlicer GitHub issues report that `ensure_vertical_shell_thickness = all` causes a visible "hull line" artefact on prints (e.g. 3DBenchy). `none` overrides top/bottom shell counts entirely. `moderate` is the safer middle ground. The legacy `enabled` value maps to one of the modern modes. <https://github.com/SoftFever/OrcaSlicer/issues/4682> · <https://github.com/SoftFever/OrcaSlicer/issues/4474>

[^precise-wall]: OrcaSlicer "Precise Wall" feature — sets the outer-to-inner-wall overlap to zero to prevent the inner wall pushing out the outer, improving dimensional accuracy. Recommended for use with `inner-outer-inner` wall ordering. **Note**: one community source incorrectly claims Klipper users should disable Precise Wall and Arc Fitting because Klipper doesn't support G2/G3 — this is **incorrect**. Klipper has supported arc commands via `[gcode_arcs]` since 2020 and our `printer.cfg` has it enabled. <https://github.com/SoftFever/OrcaSlicer/discussions/831> · <https://github.com/SoftFever/OrcaSlicer/discussions/4697>

[^bridge-flow]: OrcaSlicer Bridging Wiki and community guidance: reducing bridge flow ratio slightly (typical 0.9-0.95) prevents sag on unsupported spans. PLA is well-behaved but the 5 % reduction is the standard tuning practice. <https://github.com/SoftFever/OrcaSlicer/wiki/quality_settings_bridging>

[^internal-bridge]: Multiple OrcaSlicer GitHub issues report that the default `internal_bridge_flow = 1.5` over-flows and causes lumps that protrude above the current layer, sometimes causing nozzle crashes. Modern community consensus is to set internal bridge flow to 1.0 (or 0.9 for severe cases). The actual flow used is the product of `bridge_flow × internal_bridge_flow × filament_flow_ratio`. <https://github.com/SoftFever/OrcaSlicer/issues/4683> · <https://github.com/SoftFever/OrcaSlicer/issues/5097> · <https://github.com/SoftFever/OrcaSlicer/discussions/11961>

[^ironing-angle]: OrcaSlicer Issue #10834 and the Ironing Wiki recommend using a fixed angle that's offset from the solid infill direction (default 45°). Suggested values are 0° or 90°, which give a uniform direction across all top surfaces and "reduce tiger striping when reflecting light". <https://github.com/SoftFever/OrcaSlicer/issues/10834> · <https://github.com/SoftFever/OrcaSlicer/wiki/quality_settings_ironing>

[^xy-hole]: OrcaSlicer XY Hole Compensation — works by adding/subtracting a value (mm) from hole radius per layer. Material-specific (shrinkage varies by filament), so ideally a per-filament setting (open feature request to make it so). **Caveat**: GitHub Issue #8011 documents that when a hole has a *partial opening to the outside* of the part, `xy_hole_compensation` causes a visible ridge artefact inside the hole. <https://github.com/SoftFever/OrcaSlicer/issues/8011> · <https://github.com/OrcaSlicer/OrcaSlicer/issues/3990> · <https://kingroon.com/blogs/3d-print-101/understanding-orca-slicer-hole-compensation>

[^elephant-foot]: Elephant foot compensation default is 0.1 mm in Orca community consensus, range 0-1 mm. Tuned printers with accurate first-layer Z typically need *less* compensation, not more, because the elephant foot effect is mostly caused by first-layer over-squish. Alternative for severe cases: lower bed temperature 5-10 °C. <https://github.com/SoftFever/OrcaSlicer/discussions/557> · <https://help.prusa3d.com/article/elephant-foot-compensation_114487>

[^slow-down-layers]: OrcaSlicer Initial Layer Speed Wiki — `slow_down_layers` is the number of layers (after the initial layer) over which the print speed gradually ramps up linearly to full speed. Improves adhesion on tall prints with thin first features. No specific community consensus on the value; common range is 2-5. <https://github.com/SoftFever/OrcaSlicer/wiki/speed_settings_initial_layer_speed>

[^adaptive-layer]: Adaptive layer height — community is divided. Theoretically improves curved-surface quality and reduces print time by varying layer height per layer based on geometry. **Practical issue**: the layer-height transitions create visible banding on vertical walls because the Z stops change with each transition. Best used only when the geometry change is structural rather than aesthetic. VzBot bundled keeps it off. <https://www.obico.io/blog/orca-slicer-adaptive-and-variable-layer-height-guide-smoother-3d-prints/>

[^line-width-bambu]: Bambu Lab Wiki on line width — community-canonical guidance for 0.4 mm nozzle line widths. Outer wall: 105-120 % (0.42-0.48 mm) for clean overhangs and detail. Inner wall: ≥ 120 % for structural strength. Top surface: 100-105 %. Sparse infill: up to 150 %. General range: 75-150 % of nozzle diameter. <https://wiki.bambulab.com/en/software/bambu-studio/parameter/line-width>

[^seam-position]: OrcaSlicer Seam Wiki and community discussion — "For most prints, Aligned or Back gives the cleanest result." `Random` is recommended only on organic shapes. `Nearest` minimises travel time but spreads visible seam scars. `Aligned_back` is the quality-best choice for objects with a clear "back" face. <https://github.com/SoftFever/OrcaSlicer/wiki/quality_settings_seam> · <https://www.obico.io/blog/orcaslicer-seam-settings/> · <https://help.prusa3d.com/article/seam-position_151069>

[^scarf-guide]: "Better Seams: An Orca Slicer Guide to Using Scarf Seams" — comprehensive Printables tutorial by community contributor Adam L (no relation to repo owner). Tested against Prusament Simply Green PLA across hundreds of experiments. Confirms default 20 mm length and 10 steps work well in most cases; recommends avoiding scarf placement on overhangs via seam painting where possible. <https://www.printables.com/model/783313-better-seams-an-orca-slicer-guide-to-using-scarf-s>
