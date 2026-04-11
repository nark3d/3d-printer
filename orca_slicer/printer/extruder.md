# OrcaSlicer Printer Settings — Extruder tab

> One file per tab. See [`README.md`](README.md) for the index of all tab documents and shared hardware context.
>
> The Extruder tab covers retraction, wipe, z hop, and per-extruder geometry. **For us this tab is dominated by retraction tuning** — the Orbiter 2.5 with its 7.5 : 1 gear ratio and ~0.06 mm measured backlash has very different retraction needs than typical direct-drive extruders.

---

## ⚠️ Critical finding flagged at the top

Your current Extruder profile has **the exact "do not do this" combination from Adam L's scarf seam guide**:

| Setting | Currently | Adam L's rule |
|---|---|---|
| `retract_when_changing_layer` | **`false` (0)** | "Retract on layer change ON + wipe while retracting ON. The combination of both OFF causes problems." |
| `wipe` (wipe while retracting) | **`false` (0)** | Same |

**Both are off.** This is the failure mode the [field notes document](../process/seam_diagnosis_field_notes.md) explicitly identifies as bad — the wall ends with full nozzle pressure and no cleanup, the wall starts with pressure deficit after retraction, and the cumulative effect is visible seam artefacts that look gorge-like.

The 2024 known-good profile referenced in the field notes presumably had these set correctly. Either they got changed and never reverted, or they were never correctly configured in this profile. Either way: **strongly recommend turning both ON**.

---

## Retraction

| Setting | OrcaSlicer default (Klipper base) | Researched value | Our value | Comments |
|---|---|---|---|---|
| **Retraction length** | 0.8 mm | **0.4-0.8 mm** for Orbiter 2.5 PLA [^orbiter-retraction] | **1.0 mm** | Empirically calibrated. Voron/RatRig community range for Orbiter 2.5 + PLA is 0.4-0.8 mm; our 1.0 is on the high side but defensible for melt-zone pressure relief (Orbiter backlash is only 0.06 mm so it's not the constraint). **DECISION: no change** — empirically calibrated, leave alone. If the enabled-wipe change causes stringing, reconsider with an OrcaSlicer Retraction Test. |
| **Z hop when retracting** | 0.4 mm (Klipper base) | 0.2-0.4 mm | **0.4 mm** *(inherited)* | Lifts nozzle 0.4 mm during retraction travel to avoid scarring printed surfaces. Klipper base default is 0.4 — reasonable for our low-Z-velocity setup. **DECISION: no change** — default is correct. |
| **Z hop type** | `Normal Lift` (Klipper base) | `Spiral Lift` (smoother) [^z-hop-type] | **`Spiral Lift`** | Spiral Lift corkscrews up while moving (no pause), critical for our `max_z_velocity: 5 mm/s` where vertical lift pauses are visible (~80 ms). **DECISION: no change** — override is correct. |
| **On surfaces** (`retract_lift_enforce`) | `All Surfaces` | `Top Only` for quality (avoids scarring outer walls during travel) | **`Top Only`** | Z hop only on travel over printed top surface — avoids unnecessary hops over bed travel. **DECISION: no change** — override is correct for quality goal. |
| **Retract on layer change** | `true` (1) (Klipper base) | **`true` (1)** [^layer-change-retract] | **`false` (0)** | **HALF OF ADAM L'S "DO NOT" COMBO.** Combination of this OFF + `wipe` OFF causes seam gorges (see [field notes](../process/seam_diagnosis_field_notes.md)). **DECISION: CHANGE to `1`** — critical fix. |
| **Wipe while retracting** | `true` (1) (Klipper base) | **`true` (1)** [^wipe-while-retracting] | **`false` (0)** | **HALF OF ADAM L'S "DO NOT" COMBO.** Same as above — both off is the failure mode. **DECISION: CHANGE to `1`** — critical fix. |
| **Retract on filament swap** | (default) | (default) | (default) | Only relevant for multi-material printing. **DECISION: no change** — N/A for single-extruder. |
| **Retraction min travel** | 1 mm (Klipper base) | **1-3 mm** for direct drive | **3 mm** | Below this travel, no retraction happens. 3 mm avoids retracting on every tiny infill move. Orbiter could handle smaller, but 3 mm is a time-saving choice. **DECISION: no change** — override defensible for print time on infill-heavy models. |
| **Retract amount before wipe** | 70 % (Klipper base) | 70 % *(default)* | (default 70 %) | Fraction of retraction that happens before the wipe starts. Default 70 % is well-tuned. **DECISION: no change** — will take effect once wipe is enabled. |
| **Retraction speed** | 30 mm/s (Klipper base) | **30-60 mm/s** typical, up to 120 mm/s for Orbiter | **120 mm/s** | Aggressive but defensible for Orbiter 2.5's 7.5:1 gear ratio and low backlash. Empirically tested. **DECISION: no change** — calibrated, leave alone. |
| **Deretraction speed** | 30 mm/s (Klipper base) | **30-60 mm/s** typical, faster acceptable | **120 mm/s** | Same as retraction speed. **DECISION: no change** — calibrated. |
| **Wipe distance** | (varies, often 2 mm) | **2 mm** standard when wipe is enabled [^wipe-distance-footgun] | (inherited, effectively 2 mm) | Only meaningful when `wipe` is enabled. The Klipper-common parent sets this implicitly via the wipe mechanism. **DECISION: SET EXPLICITLY to `2` mm** — makes the value visible in the profile JSON and matches the field-notes recommendation. Will take effect once `wipe: 1` is applied. |

---

## Lift before retract

These are newer OrcaSlicer settings that may or may not appear in your current version.

| Setting | OrcaSlicer default | Researched value | Our value | Comments |
|---|---|---|---|---|
| **Lift before retract distance** | 0 (off) | 0 (default — only useful for stringy filaments) | (default) | Non-zero means lift Z *before* retracting, instead of simultaneously. Useful for very stringy filaments (PETG, TPU) where pressure needs to relax before withdrawal. PLA doesn't need it. **DECISION: no change** — default 0 is correct for PLA. |

---

## Travel & toolhead

| Setting | OrcaSlicer default | Researched value | Our value | Comments |
|---|---|---|---|---|
| **Extra length on restart** | 0 (Klipper base) | 0 *(default)* | (default 0) | Extra extrusion after de-retraction to compensate for missed material. Only needed for very long retractions or specific filaments. **DECISION: no change** — default 0 is correct for PLA at 1 mm retraction. |
| **Extra length on restart for filament toolchange** | 0 | 0 | (default 0) | Same but for multi-material filament swaps. **DECISION: no change** — N/A for single-extruder. |
| **Retraction length on filament toolchange** | 2 (Klipper base) | 2 *(default)* | (inherited) | Retraction length during a multi-material tool change. **DECISION: no change** — N/A for single-extruder. |

---

## Wipe

This subsection covers the toolhead-side wipe behaviour. The slicer-side wipe (`wipe_on_loops`) is in the Quality tab.

| Setting | OrcaSlicer default | Researched value | Our value | Comments |
|---|---|---|---|---|
| **Wipe** (wipe while retracting) | `true` (1) (Klipper base) | **`true` (1)** [^wipe-while-retracting] | **`false` (0)** | Same setting as the Retraction section above — listed again here for UI visibility. **DECISION: CHANGE to `1`** (already flagged in Retraction section — fixes half of the Adam L "do not" combination). |

---

## Nozzle / hot end

| Setting | OrcaSlicer default | Researched value | Our value | Comments |
|---|---|---|---|---|
| Nozzle diameter | 0.4 mm (parent profile) | **0.4 mm** | 0.4 mm *(inherited)* | Matches the physical hardened steel nozzle. **DECISION: no change** — matches hardware. |
| Nozzle volume | 0 (default) | 0 *(default — only used for multi-material)* | (default 0) | Estimated melt-chamber volume for multi-material purge calculations. **DECISION: no change** — N/A for single-extruder. |
| Extruder offset | 0/0 | 0/0 *(single extruder)* | (default 0/0) | XY offset of this extruder from printer origin. Always 0/0 for single-extruder. **DECISION: no change** — correct. |

---

## Summary of recommended Extruder tab changes

### Strongly recommended (fixes the Adam L "do not" combination)

| Setting | Currently | Recommend | Reason |
|---|---|---|---|
| `retract_when_changing_layer` | **`false` (0)** | **`true` (1)** | Adam L's avoid list — combination of this off + wipe off causes seam gorges per the [field notes](../process/seam_diagnosis_field_notes.md). Klipper base default is on. Turn ON. |
| `wipe` | **`false` (0)** | **`true` (1)** | Same — half of the bad combination. Klipper base default is on. Turn ON. |

These two changes together fix the bad combination. Both changes should be made simultaneously.

### Optional refinements (small tuning, not blocking)

| Setting | Currently | Could refine to | Reason |
|---|---|---|---|
| `retraction_length` | 1.0 mm | 0.5-0.8 mm | Voron/RatRig community range for Orbiter 2.5 + PLA. Run OrcaSlicer's retraction test to find the optimum. The 0.06 mm Orbiter backlash means low values work. |
| `wipe_distance` | (inherited) | 2 mm — *but only after `wipe: true` is set* | Currently inert because wipe is off. After enabling wipe, 2 mm is the standard distance |

### Confirmed correct (no change needed)

- `nozzle_diameter: 0.4 mm` — matches hardware
- `z_hop: 0.4 mm` — reasonable for our Z velocity
- `z_hop_types: Spiral Lift` — smoothest, important on slow-Z machines
- `retract_lift_enforce: Top Only` — quality choice
- `retraction_minimum_travel: 3 mm` — defensible Orbiter setting
- `retract_before_wipe: 70 %` — Klipper base, well-tuned
- `retraction_speed: 120 mm/s` — Orbiter can handle this; defensible
- `deretraction_speed: 120 mm/s` — same

### Cross-reference with Quality tab

The Quality tab document ([`../process/quality_PLA.md`](../process/quality_PLA.md)) covers `wipe_on_loops` separately — that's the **process-side** wipe (per-loop wipe behaviour), which is independent of the **printer-side** `wipe` setting covered here. Adam L's guidance:

| Setting | When using scarf seams | When using traditional seams |
|---|---|---|
| `wipe_on_loops` (Quality / process) | **OFF** | ON |
| `wipe` (Extruder / printer) | **ON** | ON |
| `retract_when_changing_layer` (Extruder / printer) | **ON** | ON |

Both `wipe` and `retract_when_changing_layer` should be **ON regardless of seam type**. The conflict is only with `wipe_on_loops` (the process-side per-loop wipe).

---

## What changes after fixing the wipe + retract on layer change?

**Visible quality improvement on prints with normal seams** (non-scarf'd corners). The current profile produces seam artefacts of the type the field notes describe — gorges at the seam location due to no pressure relief. After enabling both:

- The wall ends with a wipe move that bleeds off pressure cleanly
- Layer transitions retract first, eliminating ooze during the layer change
- Seam regions become much cleaner (less bump, less gorge)

This is a **fix for an active problem**, not a refinement. If you have any prints with visible seam scars, this is likely the fix.

---

## References

[^orbiter-retraction]: Orbiter 2.5 retraction values — Manufacturer recommendation 1.2 mm; Voron/RatRig community range 0.4-0.8 mm for PLA; measured backlash 0.06 mm. Direct drive geared extruders need much less retraction than Bowden setups. Cross-reference: [`../process/quality_PLA.md`](../process/quality_PLA.md) `[^orbiter25]` footnote. <https://www.orbiterprojects.com/orbiter-v2-5/>

[^z-hop-type]: OrcaSlicer Z hop type options — `Normal` (vertical lift then horizontal move, slowest), `Spiral` (corkscrew up while moving horizontally, smoothest, no pause), `Slope` (combined Z + XY linear), `Auto` (slicer picks). Spiral is the best choice on printers with low Z velocity because it eliminates the pause at the lift point. Our Klipper `max_z_velocity: 5 mm/s` makes this matter — at 5 mm/s, a 0.4 mm vertical lift takes 80 ms which is visibly long during fast XY travel.

[^layer-change-retract]: OrcaSlicer `retract_when_changing_layer` — when enabled, the slicer emits a retraction at the start of each layer change (just before the Z hop and travel to the next layer). Combined with `wipe`, prevents ooze during the layer transition. **Adam L's "Better Seams" guide explicitly identifies the combination of this OFF and `wipe` OFF as a failure mode** — the wall ends with full pressure, the next layer starts with a pressure deficit, and the cumulative effect is visible seam artefacts. Cross-reference: [`../process/seam_diagnosis_field_notes.md`](../process/seam_diagnosis_field_notes.md). <https://www.printables.com/model/783313-better-seams-an-orca-slicer-guide-to-using-scarf-s>

[^wipe-while-retracting]: OrcaSlicer printer-side `wipe` setting (separate from process-side `wipe_on_loops`). When enabled, the nozzle moves inward toward the part during the retraction sequence, dragging off any residual material on the nozzle. Should be **on for both scarf and traditional seams**. The conflict Adam L warns about is the *combination* of this off plus `retract_when_changing_layer` off — together they leave the wall with no pressure relief at the seam. <https://github.com/SoftFever/OrcaSlicer/wiki/printer_extruder>

[^wipe-distance-footgun]: The OrcaSlicer wipe distance setting **does nothing** unless either `wipe` (extruder tab) or `wipe_on_loops` (quality tab) is enabled. The slicer UI accepts a value but emits no wipe gcode if both flags are off. Identified in the field notes document as the cause of a "1 mm gorge through the entire wall" failure mode where the wipe distance was set to 2 mm but no wipe moves were ever generated. Cross-reference: [`../process/seam_diagnosis_field_notes.md`](../process/seam_diagnosis_field_notes.md) Case 2.
