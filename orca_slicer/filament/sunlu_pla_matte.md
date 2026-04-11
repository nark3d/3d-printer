# SUNLU PLA Matte 1.75 mm — filament profile

> One file per physical filament. See [`README.md`](README.md) for shared hardware context and the column format.
>
> **Product**: SUNLU 3D Printer Filament PLA Matte 1.75 mm (any colour, 1 kg spool) — the standard (not "High-Speed") matte PLA variant.
>
> **Our profile name**: `Sunlu PLA Matte @MyKlipper 0.4 nozzle.json`
> **Bundled parent**: OrcaSlicer ships a `SUNLU PLA Matte @System` profile derived from `SUNLU PLA Matte @base` — our current profile **does not inherit from it** (inherits: "" — it's a flat base profile duplicated then edited). This is noted below as a cleanup opportunity.

---

## ⚠️ Findings flagged at the top

| Item | Status | Note |
|---|---|---|
| `filament_vendor` is set to **"eSUN"** | 🐛 **Bug** | Cosmetic — the profile was clearly cloned from an eSUN profile and the vendor field wasn't updated. Should be "SUNLU" |
| `filament_z_hop` + `filament_z_hop_types` override the Spiral Lift default to **Normal Lift 0.2 mm** | ⚠️ **Unintentional regression?** | The extruder tab uses Spiral Lift 0.4 mm deliberately (see [`../printer/extruder.md`](../printer/extruder.md)). This filament-level override silently downgrades to Normal Lift when this filament is loaded. Unclear if deliberate |
| `filament_max_volumetric_speed: 24` | ⚠️ **Aggressive** | Orca bundled default for matte PLA is **12** — ours is 2×. Defensible *only* if a max flow test on this hotend + this filament returned ≥ 24 without under-extrusion. Most matte PLA community values are 10-16 mm³/s [^matte-flow-cap] |
| `filament_vendor`, `filament_density` drift from bundled SUNLU profile | ℹ️ Cosmetic | Our density 1.23 vs bundled 1.3; our vendor "eSUN" vs bundled "SUNLU". The bundled @base profile would be a cleaner starting point |
| `pressure_advance: 0.045` | ✅ | Matches what we calibrated empirically — documented in [`../process/seam_diagnosis_field_notes.md`](../process/seam_diagnosis_field_notes.md) |
| `filament_flow_ratio: 0.98` | ✅ | Matches the Orca bundled default for SUNLU PLA Matte AND the community-calibrated range (0.96-0.98) [^sunlu-flow] |
| `filament_retraction_length: 0.5 mm` (overrides printer-tab 1.0 mm) | ✅ | Good Orbiter 2.5 value — matches Voron/RatRig community range 0.4-0.8 mm |

**Most important**: the Z hop override and the vendor field bug. The max volumetric speed of 24 is high but **may** be legitimate.

---

## Product identification

From SUNLU's own documentation and retailer spec sheets [^sunlu-official]:

| Parameter | SUNLU spec |
|---|---|
| Print temperature | **200-230°C** (recommended 205-215°C) |
| Bed temperature | **50-65°C** (heated bed optional) |
| Print speed | **30-90 mm/s** (basic matte variant; 600 mm/s for High-Speed Matte) |
| Diameter | 1.75 mm ± 0.02 mm |
| Density | ~1.24-1.30 g/cm³ (not explicitly stated on most product pages) |
| Spool | 1 kg net, 200 mm outer diameter |

Note: SUNLU sells both a **standard Matte PLA** and a **High-Speed Matte PLA**. We are documenting the *standard* one. If your spool is labelled "High-Speed Matte PLA" (600 mm/s capable), the recommended print temps shift up (210-260°C depending on speed) and the max volumetric speed is higher.

---

## Temperature

| Setting | OrcaSlicer default (SUNLU @base) | Researched value | Our value | Comments |
|---|---|---|---|---|
| **Nozzle temperature** (layer ≥ 2) | 220°C (inherited from `fdm_filament_pla`) | **210-220°C** for standard matte PLA [^sunlu-official] | **220°C** | Middle of SUNLU's spec range; community sweet spot. **DECISION: no change** — 220°C is defensible for our quality-first profile. |
| **Nozzle temperature initial layer** | 220°C (inherited) | 215-220°C | **220°C** | Some users drop 5°C on first layer for better adhesion, but not a problem at 220. **DECISION: no change** — only revisit if first-layer elephant foot appears. |
| **Nozzle range low** | 205°C (SUNLU @base) | 200°C | **200°C** | UI slider lower bound only (not enforced). **DECISION: no change** — override 200°C is fine. |
| **Nozzle range high** | 245°C (SUNLU @base) | 230°C (SUNLU official) | **230°C** | UI slider upper bound, tightened to match SUNLU official max. **DECISION: no change** — override is sensible. |
| **Hot plate temp** (textured PEI) | 55°C (inherited from `fdm_filament_pla`) | **60°C** for textured PEI adhesion | **60°C** | Standard PLA-on-PEI. **DECISION: no change** — override is correct. |
| **Hot plate temp initial layer** | 55°C (inherited) | 60°C | **60°C** | Same. **DECISION: no change**. |
| **Textured plate temp** | 55°C (inherited) | 60°C | **60°C** | Same. **DECISION: no change**. |
| **Cool plate temp** | 35°C (inherited from `fdm_filament_pla`) | 35°C | **60°C** | We don't use a cool plate. Value is cosmetic. **DECISION: no change for now** — only matters if you switch plates; leave at 60 to match other PLA bed values. Could normalise to 35 as a cleanup but no functional impact. |
| **Supertack plate temp** | 45°C (inherited from `fdm_filament_common`) | 45°C | **35°C** | N/A — we don't use a Supertack plate. **DECISION: no change** — cosmetic. |
| **Temperature vitrification** | 54°C (SUNLU @base) | 50-55°C for matte PLA [^sunlu-official] | **60°C** | Matte PLA's actual Tg is 50-55°C. 60°C is optimistic — affects the slicer's overheat warnings only, no slicing impact. **DECISION: CHANGE to `55`** — makes the warning threshold honest without affecting any print output. |

### Note on "initial layer = layer" temperatures

Some filament profiles use a slightly lower initial layer temperature (e.g. 215°C initial, 220°C layer) to reduce stringing off the prime line and improve first-layer finish. **Ours is 220°C/220°C** which is acceptable but could be tuned.

---

## Flow / material properties

| Setting | OrcaSlicer default (SUNLU @base) | Researched value | Our value | Comments |
|---|---|---|---|---|
| **Flow ratio** | **0.98** (SUNLU @base) | **0.96-0.98** after calibration [^sunlu-flow] | **0.98** | Empirically calibrated; matches bundled default and community range. **DECISION: no change** — calibrated value. |
| **Filament density** | **1.3** g/cm³ (SUNLU @base) | 1.24 g/cm³ standard PLA | **1.23** | Affects spool weight estimates only, no slicing impact. Standard PLA is 1.24. **DECISION: CHANGE to `1.24`** — matches SUNLU implicit spec more accurately. Cosmetic but clean. |
| **Max volumetric speed** | **12 mm³/s** (SUNLU @base) | **10-16 mm³/s** for matte PLA | **24 mm³/s** | Flagged finding: 24 is 2× the bundled matte PLA default. Matte PLA is filler-limited and usually caps at 12-16 mm³/s. **DECISION: no change for now** — needs a Max Volumetric Speed calibration print to verify. If you haven't run that test, flag it on NEXT_STEPS. Don't guess a number without the test. |
| **Filament cost** | 25.99 (SUNLU @base, USD) | — | **15** | UK GBP cost. **DECISION: no change** — personal accounting value. |
| **Filament diameter** | 1.75 mm | 1.75 mm | **1.75 mm** | Standard. **DECISION: no change** — matches hardware. |
| **Filament type** | PLA | PLA | **PLA** | Correct. **DECISION: no change**. |
| **Filament vendor** | SUNLU (SUNLU @base) | SUNLU | **eSUN** | 🐛 Cloned profile never updated. **DECISION: CHANGE to `"SUNLU"`** — cosmetic bug fix. |

### On the max volumetric speed question

Matte PLA specifically is harder to push at high flow because:

1. **Filler content** — calcium carbonate or similar matting agents increase viscosity and reduce the rate at which the melt can reach steady state
2. **Thermal inertia** — fillers absorb heat differently from pure PLA, meaning the melt temperature at the nozzle isn't quite as stable at high flow
3. **Surface finish dependency** — matte PLA looks matte because of slight under-extrusion at the surface layer. Push it too hard and the characteristic matte finish degrades to striation and patchy gloss

The community consensus for matte PLA on high-flow hotends is **12-16 mm³/s for best surface quality**, with some users going up to 20 mm³/s for bulky / structural prints where finish doesn't matter. **24 mm³/s is outside that window.**

**Specific recommendation**: run OrcaSlicer's Max Volumetric Speed calibration test with this filament on this printer. If it passes 24 mm³/s cleanly (consistent lines, no glossy-to-matte transition, no under-extrusion), leave it. If it starts failing around 15-18 (which is the typical matte PLA ceiling), drop to that value.

---

## Cooling / fan

Matte PLA needs **strong cooling** to develop its characteristic matte finish — the filler particles form a slightly textured surface as the melt cools, and if the cooling is too slow, the surface glosses up instead. This is why our fan strategy is so aggressive.

| Setting | OrcaSlicer default (`fdm_filament_pla`) | Researched value | Our value | Comments |
|---|---|---|---|---|
| **Close fan first X layers** | 1 | 1-2 | **1** | Fan off for layer 1 only (bed adhesion). **DECISION: no change** — standard. |
| **Full fan speed layer** | 3 | 3 | **3** | Fan ramp-up to 100% by layer 3. **DECISION: no change** — standard. |
| **Fan min speed** | 100 | 100 for matte PLA | **100** | Always 100% once fan is on — correct for matte PLA finish. **DECISION: no change**. |
| **Fan max speed** | 100 | 100 | **100** | Same. **DECISION: no change**. |
| **Fan cooling layer time** | 100 | 100 | **2** | Functionally moot (fan_min_speed is already 100%) but looks confusing. **DECISION: no change for now** — revert to default 100 is a cleanup option but has zero functional impact. Skip in this pass. |
| **Slow down for layer cooling** | 1 (enabled) | 1 | **1** | Slow down if layer too short. **DECISION: no change** — standard. |
| **Slow down layer time** | 4 | 4-8 | **2** | Override "almost never slows down". Defensible if fan cooling is sufficient. **DECISION: no change** — empirically set, leave alone. |
| **Slow down min speed** | 10 | 10-20 | **40** | Override "never drop below 40 mm/s even if layer is too hot". Aggressive but empirical. **DECISION: no change** — leave alone. |
| **Additional cooling fan speed** | 70 | 70 | **70** | Auxiliary part cooling fan. **DECISION: no change** — default. |
| **Overhang fan speed** | 100 | 100 | **100** | **DECISION: no change** — default. |
| **Overhang fan threshold** | 50% (`fdm_filament_pla`) | 25-50% | **25%** | Override to more aggressive overhang cooling. Good for matte PLA's droopier overhangs. **DECISION: no change** — override is correct. |
| **Enable overhang bridge fan** | 1 | 1 | **1** | **DECISION: no change** — standard. |
| **Internal bridge fan speed** | -1 (use main fan speed) | -1 | **-1** | **DECISION: no change** — default. |
| **Ironing fan speed** | 0 | 0 | **0** | Fan off during ironing so layer re-melts cleanly. **DECISION: no change** — correct. |
| **Reduce fan stop/start freq** | 1 | 1 | **1** | Smooths fan transitions. **DECISION: no change** — standard. |

### Cooling summary

Our cooling strategy is **aggressive cooling with minimal slow-down intervention**:

- Fan always 100% once layer ≥ 3
- Overhang threshold tighter than default (25% vs 50%)
- Slow-down almost never kicks in (layer time threshold 2 s, min speed floor 40 mm/s)

**This is the right shape** for matte PLA if you trust the fan to keep up. On very small top features or tall thin objects, consider loosening the slow-down parameters slightly (e.g. `slow_down_layer_time: 4`, `slow_down_min_speed: 20`) to give the print time to cool.

---

## Retraction (filament-side overrides)

These overrides take effect **only when this filament is loaded**. They override the printer-tab values from [`../printer/extruder.md`](../printer/extruder.md).

| Setting | OrcaSlicer default | Researched value | Our value | Comments |
|---|---|---|---|---|
| **Filament retraction length** | nil (use printer tab) | **0.4-0.8 mm** for Orbiter 2.5 PLA | **0.5 mm** | Override to 0.5 mm (printer tab is 1.0). In Orbiter community range. **DECISION: no change** — override is correct. |
| **Filament retraction min travel** | nil | 1-3 mm | **0.5 mm** | Override to 0.5 mm (very short) — more retractions but cleaner features. Defensible for matte PLA stringing sensitivity. **DECISION: no change** — override is defensible. |
| **Filament retract restart extra** | nil | 0 | **0.03 mm** | Small priming move after de-retract for pressure continuity. Empirical tweak. **DECISION: no change** — override is defensible. |
| **Filament retraction speed** | nil | — | **nil** (inherits 120 mm/s from printer tab) | **DECISION: no change** — inherits correctly. |
| **Filament deretraction speed** | nil | — | **nil** | **DECISION: no change** — inherits correctly. |
| **Filament wipe** | nil | See [`../printer/extruder.md`](../printer/extruder.md) | **nil** | Inherits from printer tab. Once printer-tab `wipe` is changed to 1 (per extruder.md decision), this filament benefits automatically. **DECISION: no change** — inherit is correct. |
| **Filament wipe distance** | nil | 2 mm (when wipe is enabled) | **2 mm** | Override to 2 mm (correct value when wipe is on). **DECISION: no change** — override is correct. |
| **Filament retract before wipe** | nil | 70% | **nil** | **DECISION: no change** — inherits correctly. |
| **Filament retract when changing layer** | nil | See [`../printer/extruder.md`](../printer/extruder.md) | **nil** | Inherits from printer tab. Once printer-tab is changed to 1 (per extruder.md decision), benefits automatically. **DECISION: no change** — inherit is correct. |
| **Long retractions when cut** | 0 | — | **0** | Multi-material filament cutter feature. **DECISION: no change** — N/A. |
| **Retraction distance when cut** | nil | — | **10** | Same. **DECISION: no change** — N/A. |

---

## Z hop (filament-side overrides)

⚠️ **Flagged section.** These overrides change the Z hop behaviour from the printer-tab defaults.

| Setting | Printer tab value | Researched value | Our filament value | Comments |
|---|---|---|---|---|
| **Filament z hop** | 0.4 mm | 0.2-0.4 mm | **0.2 mm** | Overrides printer-tab 0.4 mm. Unclear if deliberate. **DECISION: REMOVE override** — let the filament inherit the printer-tab 0.4 mm. Pairs with the Z hop type removal below. |
| **Filament z hop types** | Spiral Lift | Spiral Lift (see printer tab reasoning) | **Normal Lift** | 🚨 Silent regression — overrides extruder-tab Spiral Lift to Normal Lift, undoing the reasoning about Klipper's low Z velocity making vertical-lift pauses noticeable. **DECISION: REMOVE override** — let the filament inherit Spiral Lift from the printer tab. |

### Why this override exists (speculation)

Two plausible reasons:

1. **Copy from older profile** — an older version of the profile used Normal Lift before we discovered Spiral Lift was better; the filament profile wasn't updated when the printer profile was
2. **Deliberate override** — maybe Spiral Lift caused some issue with this specific filament (unlikely; Spiral Lift is filament-agnostic)

**Recommend removing both `filament_z_hop` and `filament_z_hop_types` overrides** so the filament inherits from the printer tab. Unless there's a specific reason we've forgotten, the printer-tab Spiral Lift 0.4 mm is the better choice.

---

## Pressure advance

| Setting | OrcaSlicer default | Researched value | Our value | Comments |
|---|---|---|---|---|
| **Enable pressure advance** | 0 | 1 | **1** | Emits `SET_PRESSURE_ADVANCE` at print start. **DECISION: no change** — override is correct. |
| **Pressure advance value** | 0 | **0.02-0.08** for direct drive, 0.04 safe default [^pa-direct-drive] | **0.045** | Empirically calibrated — matches field notes. **DECISION: no change** — calibrated value. |
| **Adaptive pressure advance** | 0 | 0 (experimental) | **0** | Experimental feature. **DECISION: no change** — leave off. |
| **Adaptive PA model** | (empty) | (empty) | `0,0,0\n0,0,0` | Placeholder. **DECISION: no change** — N/A. |
| **Adaptive PA bridges** | 0 | 0 | **0** | **DECISION: no change** — N/A. |
| **Adaptive PA overhangs** | 0 | 0 | **0** | **DECISION: no change** — N/A. |

### Cross-reference with Klipper PA

Our profile emits `SET_PRESSURE_ADVANCE ADVANCE=0.045` during print start (when `enable_pressure_advance=1` and `pressure_advance=0.045`). Klipper then applies this value for this filament's print, overriding whatever is in `printer.cfg`.

The Klipper `[extruder]` section in `printer.cfg` has its own `pressure_advance` value (defaults are set at Klipper config time, typically ~0.04 from earlier tuning). The slicer-emitted value takes effect during the print but reverts to the `printer.cfg` value at print end — so if you load a filament without PA set in the slicer, it'll use whatever is in `printer.cfg`.

**Recommended**: keep `enable_pressure_advance: 1` with the empirically calibrated value per filament, so the slicer is the source of truth for per-filament PA.

---

## Bed adhesion

| Setting | Value for this filament | Comments |
|---|---|---|
| **Bed temperature (textured PEI)** | 60°C | Standard PLA-on-PEI |
| **First layer bed temp** | 60°C | Same |
| **Recommended plate type** | Textured PEI or smooth PEI | SUNLU matte PLA sticks reliably to both. Textured PEI gives a stronger bond and a slight texture imprint; smooth PEI gives a glossy first layer (defeats the matte purpose) |
| **Purge / prime line** | Use PRIME_LINE (or KAMP's LINE_PURGE) — see [`../printer/machine_gcode.md`](../printer/machine_gcode.md) | Matte PLA's first few mm of extrusion after load can be ragged; a generous prime line helps |
| **Brim / skirt** | See [`../process/others_PLA.md`](../process/others_PLA.md) for skirt/brim recommendations | Not filament-specific |

No filament-level changes recommended. Bed temps are already correct.

---

## Air filtration / chamber

| Setting | OrcaSlicer default | Our value | Comments |
|---|---|---|---|
| `activate_air_filtration` | 0 | **0** | PLA doesn't need filtration. **DECISION: no change** — off is correct. |
| `activate_chamber_temp_control` | 0 | **0** | PLA doesn't want a heated chamber. **DECISION: no change** — off is correct. |
| `chamber_temperature` | 0 | **0** | **DECISION: no change** — off is correct. |
| `during_print_exhaust_fan_speed` | (varies) | **60** | N/A — no exhaust fan installed. **DECISION: no change** — moot. |
| `complete_print_exhaust_fan_speed` | (varies) | **80** | Same. **DECISION: no change** — moot. |

---

## Community experience and gotchas

> This section captures what other users have actually found printing SUNLU PLA Matte, as distinct from the manufacturer spec sheet. Sources: Bambu Lab community forum, Prusa forum, Reddit, Facebook 3D printing groups, and various product reviews. Individual citations in the footnotes.

### Temperature: what people actually use vs what the label says

SUNLU's product page [^sunlu-dps3ds] states 200-230°C, recommended 205-215°C. **The community converges on 210°C** as the real-world sweet spot for the standard (non-high-speed) matte variant at normal print speeds (50-150 mm/s).

| Print speed | Community-reported sweet spot | Why |
|---|---|---|
| 30-80 mm/s (slow/quality) | **205-210°C** | At low flow rates, matte PLA needs less heat to stay fluid. Lower temp = less stringing, crisper features |
| 80-150 mm/s (normal) | **210-220°C** | Our profile's 220°C sits at the top of this range — defensible for our quality-first setup |
| 150-300 mm/s (fast) | **220-230°C** | Higher flow means more heat needed to keep the melt stable. Not our use case |
| 300+ mm/s | **230-260°C** (only the "High-Speed" variant) | Requires the dedicated High-Speed variant, not the standard matte we're documenting |

**Community observation**: matte PLA is noticeably more forgiving at the *high* end of the temperature range than the low end. Users report that going below 205°C causes visible under-extrusion and weak layer bonding on the standard matte variant [^matte-low-temp-failure]. Our 220°C has good headroom.

**A counter-pattern worth noting**: the 3dpros review of SUNLU PLA Meta (a different but related SUNLU matte-style product) found that 180°C from SUNLU's own documentation caused adhesion concerns, and reviewers landed on **210°C** as the real sweet spot. This is a common theme with SUNLU matte products — the label's lower bound is often too conservative for real prints [^sunlu-meta-review]. 

**Takeaway for our profile**: 220°C is defensible. If you ever see stringing or over-extrusion artefacts, try 215°C before changing other settings.

---

### Gotcha 1 — Moisture sensitivity (matte PLA absorbs water faster than standard PLA)

**This is the single most-reported problem with SUNLU PLA Matte.** Matte PLA absorbs moisture faster than glossy PLA because the mineral filler content (calcium carbonate or similar) is mildly hygroscopic on top of PLA's own baseline moisture affinity [^wet-filament-general]. Combined with SUNLU's packaging (which is vacuum-sealed with desiccant *when shipped*, but not re-sealable after opening), this means spools that sit out for a week or two can start showing wet-filament symptoms.

**Signs your spool is wet:**

| Symptom | What's happening |
|---|---|
| **Audible popping / hissing from the nozzle** during printing | Trapped water in the filament flashes to steam as it passes through the hot end [^wet-filament-general] |
| **Tiny bubbles in the extruded line** (visible on the prime line) | Same — steam expanding inside the melt creates micro-voids |
| **Stringing that wasn't there on the first use of the spool** | Wet PLA extrudes inconsistently — pressure fluctuations cause oozing between retractions |
| **Surface gloss where there should be matte finish** | **This is the matte-specific one.** Wet matte PLA's surface texture changes — the filler particles no longer form the expected rough-surface geometry because the steam bubbles disrupt melt-side solidification. The result looks more like slightly bumpy glossy PLA |
| **Visibly weaker layer bonding** | Steam bubbles at layer interfaces reduce the effective welded area |
| **Rough, furry, or fuzzy surface** | Inconsistent flow + stringing at the filament-level |

**Prevention and cure:**

- **Store in a dry box** with fresh desiccant after opening. SUNLU sells their own dry box (ironic — they know moisture is an issue)
- **Dry before first print if the spool has been open more than 2 weeks**: 45-50°C for 4-6 hours in a dedicated dryer (SUNLU's own S1/S2/S4 dryers, Polybox, etc.)
- **Do NOT exceed 55°C when drying matte PLA** — the glass transition is only slightly above that, and the spool can deform or tangle
- **If you have no dryer**: an oven at the lowest setting with the door slightly ajar (monitor with a thermometer — must stay below 55°C) for 4 hours will recover a spool
- **Signs of dryness recovered**: popping sound disappears, stringing on a stringing test drops to near zero

**Why this matters for our profile**: our fan_min_speed: 100 cooling strategy actually helps mask some wet-filament symptoms by cooling the extrusion quickly, but it doesn't fix the underlying problem. If you see stringing on a print that used to print clean, **check the spool's moisture before touching slicer settings**.

---

### Gotcha 2 — First-layer adhesion problems straight from the vacuum pack

Multiple users report [^adhesion-batch-issues] that some batches of SUNLU PLA Matte (including the standard matte variant) **fail to stick to PEI on the first layer** despite correct bed temperature and first-layer speed. This appears to be a batch-specific QC issue — not a fundamental problem with the material.

**Hypothesised cause**: residual oil or lubricant from the filament extrusion line at SUNLU's factory. Users on the Prusa forum reported that SUNLU switched packaging (from black/white to brown) around 2022-2023 and new spools started having adhesion issues where older spools worked fine — suggesting a process change affected this.

**Diagnosis:**

- Clean PEI bed with hot water + dish soap (not IPA — soap removes oil better than alcohol)
- Dry thoroughly
- Try printing a small test. If it sticks, the bed was the problem (unrelated to the filament)
- If it *still* doesn't stick after a clean bed, the spool probably has surface contamination

**Fixes that work:**

1. **Wipe the filament** with a microfibre cloth + IPA as it feeds into the extruder. A filament cleaner attachment (sponge + IPA reservoir) is the repeatable version
2. **Use a glue stick** on the first layer only (Elmer's Washable, Pritt Stick, etc.). Unusual for PLA-on-PEI but effective
3. **Raise the bed to 65°C** for the first layer (SUNLU's official range max), then drop to 60°C for subsequent layers
4. **Slow the first layer** to 20-30 mm/s even if your normal first-layer speed is faster
5. **Run the spool through a dryer** — if the outside surface is slightly contaminated, drying at 45°C for a couple of hours can evaporate the oil

**Why this matters for our profile**: our bed temp is 60°C, which is in the middle of SUNLU's range. If you ever hit the "new spool won't stick" problem, the first thing to try is **raise to 65°C for first layer only** (a process-profile tweak, not a filament setting change).

---

### Gotcha 3 — Stringing is more common than with standard PLA (wet or not)

Even dry, matte PLA is noticeably stringier than standard PLA because:

1. **Higher melt viscosity** from the filler content changes how pressure dissipates at retraction
2. **Longer pressure relaxation time** in the melt — the nozzle continues to ooze for longer after a retract move
3. **The matte finish makes strings more visible** — tiny strings that would blend into a glossy print show up clearly on a matte surface

Community retraction recommendations [^stringing-community]:

| Extruder type | Retraction length for matte PLA |
|---|---|
| **Direct drive (Orbiter 2.5, LGX, Dragonfly)** | **0.4-0.8 mm** — our 0.5 is in range |
| **Short Bowden (< 200 mm)** | 2-3 mm |
| **Long Bowden (≥ 500 mm)** | 4-6 mm |

**Community advice**: if stringing persists despite good retraction, **drop the nozzle temperature by 5°C** before tuning retraction further. SUNLU's own guide [^sunlu-stringing-guide] explicitly recommends this as the first action.

**Why this matters for our profile**: our 0.5 mm retraction is good for the Orbiter. If you see stringing, the order of operations should be:

1. Check moisture (gotcha 1)
2. Try 215°C instead of 220°C
3. Increase retraction to 0.6-0.7 mm (still in the Orbiter range)
4. Run OrcaSlicer's Retraction Test calibration

---

### Gotcha 4 — Pressure advance is higher than standard PLA

Community and our own calibration both agree: **matte PLA needs more pressure advance than standard PLA** of the same brand. This is because the filler content changes the pressure-relaxation dynamics in the melt — the nozzle takes slightly longer to reach steady state after a direction change or retraction.

| Filament type | Typical direct-drive PA |
|---|---|
| Standard PLA (glossy) | **0.025-0.040** |
| Matte PLA | **0.035-0.055** |
| Silk PLA | 0.025-0.035 (silk additives reduce viscosity) |
| PLA+ / PLA Pro | 0.030-0.045 |
| Carbon Fibre PLA | 0.050-0.080 |

**Our value**: 0.045 — sits near the middle of the matte PLA range, and specifically matches the field-notes calibrated value for Sunlu Matte PLA [^field-notes-pa]. **This is correct.**

**Gotcha**: if you swap between standard PLA and matte PLA without updating PA, the matte PLA will ghost at corners (visible "lead-in" ripples where the print started pressure-building late). Always make sure the slicer has `enable_pressure_advance: 1` with the per-filament value.

---

### Gotcha 5 — Layer adhesion is slightly weaker than standard PLA

Matte PLA parts are, all else equal, **mechanically weaker than standard PLA parts**. Measurements vary, but community testing suggests a **10-15% reduction in tensile and Z-axis (layer) strength** compared to the same brand's standard PLA [^layer-strength-matte]. The reasons:

1. **Filler particles** disrupt layer-to-layer polymer chain entanglement
2. **Lower melt temperature sweet spot** means less energy for inter-layer welding
3. **Faster cooling** (which we *want* for the matte finish) means less time for the layer to bond before the next pass

**Mitigation if you need strong matte parts:**

- **Increase wall count** (3-4 walls instead of 2) — see [`../process/strength_PLA.md`](../process/strength_PLA.md)
- **Increase infill percentage** or use gyroid infill pattern for better mechanical load distribution
- **Print hotter** (220-225°C) at the cost of slight matte finish degradation
- **Reduce fan speed** after layer 5-10 (once the layer finish is established) to allow more bonding time — this is a process-profile tweak
- **Orient critical loads along XY not Z** — this applies to all FDM but is especially important for matte PLA

**Our defaults are quality-first, not strength-first.** If you're printing something load-bearing and need strength, the field notes recommend switching to SUNLU PLA+ or eSUN PLA+ instead [^field-notes-filaments].

---

### Gotcha 6 — Abrasive? Barely. Brass nozzle is fine

**Short answer: matte PLA is NOT meaningfully abrasive.** The filler content (calcium carbonate or similar mineral) has a Mohs hardness of 2.5-3 — softer than brass (3-4). Community testing [^abrasive-matte] shows:

- **Brass nozzle**: good for 5-10 kg of matte PLA before measurable wear
- **Hardened steel nozzle**: effectively infinite life
- **Ruby / tungsten carbide**: overkill

We use a **hardened steel nozzle** already (from the profile), so this gotcha doesn't apply to us — but if you ever think "maybe I should switch to brass for better heat conduction", matte PLA is **not** the reason to stay on steel. The reason to stay on hardened steel is our toolhead is already set up for it and we occasionally run glow-in-the-dark or wood-filled PLA which *are* abrasive.

**Do NOT confuse matte PLA with carbon-fibre PLA** — CF PLA is very abrasive and will shred a brass nozzle in hours.

---

### Gotcha 7 — Nozzle ooze at print-end / between prints

Some users report that matte PLA oozes more than standard PLA when the hot end sits idle at print temperature (between prints, or during a long multi-piece session). The cause:

1. Higher melt viscosity holds some static pressure at the tip
2. Filler particles hold heat slightly longer after print-end
3. Gravity on a slightly more viscous melt equals slower but longer ooze

**Solutions:**

1. **Drop standby temp** to 170-180°C between prints (if using a print farm workflow)
2. **Retract more aggressively** at print end — our PRINT_END macro already does `G1 E-8.0 F2700` which is a **strong retract** and handles this [^print-end-retract]
3. **Don't pause-and-resume on this filament** without turning the hot end temperature down during the pause

**Our PRINT_END is already set up well for this** — see [`../printer/machine_gcode.md`](../printer/machine_gcode.md). No changes needed.

---

### Gotcha 8 — Colour consistency varies between batches

Several users report [^sunlu-batch-consistency] that SUNLU has batch-to-batch colour variation, particularly on certain colours (grey, navy, and some pastels). The matte variants are affected slightly more than the glossy variants because the filler content varies by colour (more filler = more matte, but also colour-shifts the pigment).

**Practical implication**: if you're printing a large project that spans multiple spools, try to buy them **from the same batch** (check the spool label for production date / lot number). Mid-project spool swaps can show a visible colour step.

Not a settings issue — a sourcing issue.

---

### Gotcha 9 — Not all "matte" SUNLU products are the same thing

SUNLU sells **several** matte-style PLA products, and they are not interchangeable:

| Product name | What it actually is |
|---|---|
| **SUNLU PLA Matte** (this filament) | Standard matte PLA, 30-90 mm/s, 200-230°C |
| **SUNLU High-Speed Matte PLA** | Different formulation, 50-600 mm/s, 190-260°C — optimised for high-flow printing |
| **SUNLU PLA Meta** | A specific colour series with matte-like finish but different base resin — lower viscosity, lower ideal temp [^sunlu-meta-review] |
| **SUNLU PLA Matte Dual-Color** | Dual-colour co-extruded matte PLA — visually distinct, similar settings to standard matte |
| **SUNLU PLA-CF (matte black)** | Carbon-fibre-reinforced PLA — VERY abrasive, requires hardened steel, different mechanical properties |

**Make sure you know which one you have.** The settings in this document are for **SUNLU PLA Matte** (the standard one). If your spool label says anything else, re-check before applying these settings.

---

### Summary of gotchas

| # | Gotcha | How it manifests | Fix priority |
|---|---|---|---|
| 1 | **Moisture absorption** | Popping sound, stringing, surface glosses over | **High** — dry before print if spool is > 2 weeks old |
| 2 | **First-layer adhesion from new spool** | Won't stick to PEI despite correct temps | **Medium** — batch-specific, try clean bed + 65°C first layer |
| 3 | **Stringing more common than standard PLA** | Visible strings on matte surface | **Low** — our retraction is already good |
| 4 | **Needs higher PA than standard PLA** | Corner ghosting / lead-in ripples | **Low** — our 0.045 is calibrated |
| 5 | **Slightly weaker layer bonding** | Parts fail along layer lines under load | **Informational** — use different material for strong parts |
| 6 | **NOT abrasive** | — | N/A — common misconception worth clearing up |
| 7 | **Ooze at idle** | Blob on nozzle when paused | **Low** — our PRINT_END handles it |
| 8 | **Colour batch consistency** | Visible colour step mid-project | **Sourcing issue** — buy from same batch for large projects |
| 9 | **Product name confusion** | Wrong settings for wrong variant | **Before anything else** — confirm which SUNLU product you have |

---

## Summary of recommended SUNLU PLA Matte changes

### Strongly recommended (fixes bugs and regressions)

| Setting | Currently | Recommend | Reason |
|---|---|---|---|
| `filament_vendor` | **"eSUN"** | **"SUNLU"** | Cloned profile, never updated. Cosmetic bug |
| `filament_z_hop_types` | **"Normal Lift"** | **(remove — inherit Spiral Lift from printer tab)** | Silently downgrades the extruder-tab Spiral Lift decision. See [`../printer/extruder.md`](../printer/extruder.md) for why Spiral Lift matters |
| `filament_z_hop` | 0.2 mm | **(remove — inherit 0.4 mm from printer tab)** | Same reason |

### Optional refinements (small tuning)

| Setting | Currently | Could refine to | Reason |
|---|---|---|---|
| `filament_max_volumetric_speed` | 24 mm³/s | 14-16 mm³/s (or verify with flow test) | Matte PLA typically caps at 12-16 mm³/s. 24 is high even for a Rapido 2 HF. **Run the test — don't change without verifying** |
| `temperature_vitrification` | 60°C | 55°C | Matte PLA actual Tg is 50-55°C. 60°C is optimistic |
| `filament_density` | 1.23 | 1.24 | Standard PLA density (cosmetic, affects spool weight estimates) |
| `cool_plate_temp` | 60°C | 35°C | Only matters if you ever use a cool plate |
| `fan_cooling_layer_time` | 2 | 100 (default) | Functionally moot because `fan_min_speed` is already 100% — the override is misleading but harmless |
| `slow_down_layer_time` | 2 | 4 (default) | Our override "almost never slows down" — could loosen for tall thin features |
| `slow_down_min_speed` | 40 | 20 | Same — give the slow-down logic more headroom for problem features |
| `nozzle_temperature_initial_layer` | 220°C | 215°C | Optional — slightly cooler first layer for adhesion margin |

### Confirmed correct (no change needed)

- `pressure_advance: 0.045` — empirically calibrated, matches field notes
- `filament_flow_ratio: 0.98` — matches bundled default and community range
- `nozzle_temperature: 220` — middle of SUNLU's recommended range
- `hot_plate_temp: 60` — standard PLA on PEI
- `filament_retraction_length: 0.5` — good Orbiter value
- `fan_min_speed: 100`, `fan_max_speed: 100` — aggressive cooling, correct for matte finish
- `overhang_fan_threshold: 25%` — more aggressive than default, good for matte overhangs
- `close_fan_the_first_x_layers: 1`, `full_fan_speed_layer: 3` — standard
- `filament_type: PLA` — correct
- `filament_diameter: 1.75` — correct

### Process suggestion: reparent to the bundled profile

Our current profile is a flat base (`inherits: ""`) — it doesn't walk the chain up through `SUNLU PLA Matte @base → fdm_filament_pla → fdm_filament_common`. This means:

- Any updates to the bundled SUNLU profile (e.g. if OrcaSlicer tweaks the defaults) won't propagate
- Our profile has to explicitly store every setting rather than inheriting

**Consider**: duplicate `SUNLU PLA Matte @System` from the bundled library, rename it, and then only override the values we've calibrated (PA, flow ratio, retraction, overhang threshold, fan min/max). This would produce a much smaller JSON and keep us aligned with future Orca updates.

Not urgent — the flat profile works — but it's a maintenance nicety.

---

## What changes in print behaviour after fixing the flagged items?

### After fixing the Z hop regression
- Layer changes and retractions use the smoother Spiral Lift motion
- ~40-80 ms less pause per retraction on a printer with low Z velocity
- On long prints with many retractions, shaves noticeable time off

### After tuning max volumetric speed (if needed)
- **If 24 was too high**: eliminates surface quality regression (gloss patches, under-extrusion on infill)
- **If 24 was correct**: no change

### After lowering temperature_vitrification
- The slicer's "overheat warning" becomes honest — will warn you correctly when a chamber temperature is close to softening the matte PLA
- No slicing difference

---

## References

[^rapido-realworld]: Phaetus Rapido 2 HF is specified for up to 45 mm³/s maximum volumetric flow. Real-world PLA printing typically achieves 20-24 mm³/s reliably — substantially lower than the manufacturer spec but still very high flow. One user reported printing PLA at 24 mm³/s with a Rapido HF without trouble. <https://forum.vorondesign.com/threads/maximum-volumetric-flow-rate.195/>

[^matte-flow-cap]: Matte PLA has lower max volumetric flow than standard PLA because of the mineral filler content (typically calcium carbonate or similar) that gives it the matte finish. Community consensus for matte PLA on high-flow hotends is 10-16 mm³/s for best surface quality; up to 20 mm³/s for structural prints where finish doesn't matter. Bundled OrcaSlicer SUNLU PLA Matte @base profile ships with `filament_max_volumetric_speed: 12` as the safe default. <https://forum.bambulab.com/t/volumemetric-speed-by-filament-type-and-brand/9626>

[^sunlu-official]: SUNLU's own product pages for PLA Matte 1.75 mm state: print temperature 200-230°C (recommended 205-215°C), bed temperature 50-65°C, print speed 30-90 mm/s, diameter tolerance ±0.02 mm. The "High-Speed Matte PLA" variant has extended temperature ranges (up to 260°C for 300-600 mm/s printing) but is a different product. <https://store.sunlu.com/products/over-6kg-bulk-sale-pla-matte-filament-neatly-wound-filament>

[^sunlu-flow]: Community-calibrated flow ratio values for SUNLU PLA variants cluster in the **0.96-0.98 range** after OrcaSlicer Flow Rate calibration. SUNLU PLA+ 2.0 starts at 1.00 and often drops to 0.97 after calibration. SUNLU PLA High Speed calibrates to 0.99. Matte variants typically settle at 0.98. The bundled OrcaSlicer profile uses 0.98 as the default. <https://forum.bambulab.com/t/sunlu-pla-meta-settings/23934>

[^pa-direct-drive]: Klipper pressure advance for direct drive extruders is typically in the 0.02-0.08 range, with 0.04 as a safe default. The Orbiter 2.5 specifically rarely needs more than 0.1 seconds of PA. Different filament brands and types (especially variants with additives like matte or silk) need slightly different values — matte PLA tends higher than standard PLA because the additive content slows pressure relaxation in the melt. <https://www.klipper3d.org/Pressure_Advance.html>

[^orca-matte-base]: OrcaSlicer ships a bundled `SUNLU PLA Matte @base` profile in its OrcaFilamentLibrary (located at `~/Library/Application Support/OrcaSlicer/system/OrcaFilamentLibrary/filament/SUNLU/SUNLU PLA Matte @base.json` on macOS). This bundled profile sets: `filament_density: 1.3`, `filament_flow_ratio: 0.98`, `filament_max_volumetric_speed: 12`, `temperature_vitrification: 54`, `nozzle_temperature_range_low: 205`, `nozzle_temperature_range_high: 245`, `filament_vendor: SUNLU`. It inherits from `fdm_filament_pla`.

[^matte-pla-cooling]: Matte PLA requires stronger cooling than standard PLA to develop its characteristic matte finish. The filler particles form a slightly textured surface as the melt cools — if cooling is too slow, the surface glosses up instead. Community best practice: fan 100% once past layer 1-3, overhang fan threshold 25-50%, avoid slow-down-for-cooling interventions that let the layer sit hot. <https://uk.store.sunlu.com/blogs/product-knowledge/sunlu-high-speed-matte-pla-tips-for-perfect-prints-and-applications>

### Community experience / gotchas footnotes

[^sunlu-dps3ds]: SUNLU PLA Matte 1.75 mm standard (non-high-speed) variant — full technical specification from authorised reseller: Nozzle temperature 200-230°C, Bed temperature 50-65°C, Print speed 50-100 mm/s, Diameter tolerance ±0.03 mm, Net weight 1 kg (330 m), vacuum-sealed with desiccant packaging, composition described as "mainly made of corn starch". <https://www.dps3ds.com/products/sunlu-3d-printer-filament-pla-matte-1-75mm-neatly-wound-filament-smooth-matte-finish-print-with-99-fdm-3d-printers-1kg-spool-2-2lbs-330-meters-matte-black>

[^matte-low-temp-failure]: Community reports describe under-extrusion and weak layer bonding when printing SUNLU PLA Matte below 205°C. The matte filler content increases melt viscosity, requiring more thermal energy than standard PLA to achieve the same flow rate. Users on the Bambu Lab community forum and Prusa forum consistently recommend 210°C as a minimum safe temperature for quality prints on the standard matte variant. <https://forum.bambulab.com/t/sunlu-pla-matte-filament-with-generic-pla-settings/56325>

[^sunlu-meta-review]: 3dpros filament report on SUNLU PLA Meta: SUNLU's official documentation recommended 185-195°C but the reviewers found 180°C caused adhesion concerns and landed on **210°C as the real-world sweet spot**. This is a common theme with SUNLU matte-style products — the label's lower bound is often too conservative for real prints. Note: SUNLU PLA Meta is a different product from SUNLU PLA Matte but the review is relevant context for SUNLU's general temperature-labelling pattern. <https://3dpros.com/filament/sunlu-pla-meta>

[^wet-filament-general]: PLA is mildly hygroscopic and absorbs moisture from ambient humidity over time, particularly once the vacuum seal is broken. Matte PLA variants absorb faster because the calcium-carbonate or similar filler additive is itself mildly hygroscopic on top of PLA's baseline affinity. Symptoms of wet PLA: audible popping/hissing at the nozzle (steam flash), tiny bubbles visible in extrusion, stringing that wasn't there before, surface finish degradation (matte-specific: surface glosses over where it should be matte). Recovery: dry at 45-50°C for 4-6 hours in a filament dryer. Do not exceed 55°C for matte PLA because glass transition is close to that temperature. <https://www.sovol3d.com/blogs/news/how-to-identify-wet-filament-problems-in-3d-printing>

[^adhesion-batch-issues]: Multiple forum threads describe batch-specific first-layer adhesion problems with SUNLU PLA Matte where users could not get the filament to stick to a properly prepared PEI bed despite correct temperatures and leveling. Hypothesised cause: residual oils or lubricants from the factory extrusion line. Fixes that work: (1) wipe PEI with hot water and dish soap rather than IPA, (2) wipe filament as it feeds in with IPA-soaked cloth, (3) use a glue stick as a first-layer adhesion aid, (4) raise first-layer bed temperature to 65°C (top of SUNLU's spec range), (5) slow first layer to 20-30 mm/s. Users on the Prusa forum noted a correlation between SUNLU's packaging redesign (black/white → brown) and an increase in adhesion complaints, suggesting a process change around 2022-2023. <https://forum.prusa3d.com/forum/english-forum-general-discussion-announcements-and-releases/psa-do-not-overlook-your-filament-as-a-source-of-adhesion-issues/>

[^stringing-community]: Community retraction recommendations for matte PLA vary by extruder type: direct-drive 0.4-0.8 mm (Orbiter 2.5, LGX, Dragonfly, etc.), short Bowden 2-3 mm, long Bowden 4-6 mm. SUNLU's own stringing guide recommends reducing nozzle temperature by 5°C as the first action if stringing persists despite adequate retraction. Ordering: (1) verify dry filament, (2) reduce temperature 5°C, (3) increase retraction within extruder-type range, (4) run OrcaSlicer Retraction Test calibration. <https://store.sunlu.com/blogs/3d-printing-guide/what-is-stringing-in-3d-printing-and-how-to-avoid-it>

[^sunlu-stringing-guide]: SUNLU's own stringing guide explicitly recommends reducing nozzle temperature by 5°C as the first remediation step when stringing occurs, before increasing retraction distance. This is the opposite of standard advice for non-matte PLA, reflecting matte PLA's higher temperature sensitivity. <https://store.sunlu.com/blogs/3d-printing-guide/what-is-stringing-in-3d-printing-and-how-to-avoid-it>

[^field-notes-pa]: The empirically calibrated PA value for Sunlu Matte PLA on our specific hardware (Orbiter 2.5 + Phaetus Rapido 2 HF + Klipper) is **0.045**, documented in [`../process/seam_diagnosis_field_notes.md`](../process/seam_diagnosis_field_notes.md). This sits near the middle of the community range for matte PLA on direct drive (0.035-0.055), higher than standard PLA on the same hardware (0.025-0.040).

[^layer-strength-matte]: Community mechanical testing of matte PLA vs standard PLA of the same brand consistently shows 10-15% reduction in tensile and Z-axis (inter-layer) strength. The filler particles disrupt polymer chain entanglement across layers, and the lower optimal print temperature provides less thermal energy for inter-layer welding. For load-bearing prints, matte PLA is not the right choice — switch to PLA+, PLA Pro, or PETG. <https://forum.bambulab.com/t/what-is-happening-bad-layer-adhesion-x1c-sunlu-pla/143051>

[^field-notes-filaments]: See [`../process/seam_diagnosis_field_notes.md`](../process/seam_diagnosis_field_notes.md) for per-filament calibrated values and notes on which filaments work well for which purposes on our printer. For load-bearing parts, the field notes recommend eSUN PLA+ or SUNLU PLA+ over any matte variant.

[^abrasive-matte]: Matte PLA is NOT meaningfully abrasive despite the mineral filler content. The typical filler (calcium carbonate) has Mohs hardness 2.5-3, softer than the brass used for standard nozzles (Mohs 3-4). Community testing: a brass nozzle will handle 5-10 kg of matte PLA before measurable wear, hardened steel is effectively infinite. Do NOT confuse matte PLA with carbon-fibre PLA (VERY abrasive, shreds brass nozzles in hours) or glow-in-the-dark PLA (contains strontium aluminate, genuinely abrasive). <https://www.cnckitchen.com/blog/how-much-abrasive-filaments-damage-your-nozzle>

[^print-end-retract]: Our Klipper `PRINT_END` macro (in `~/printer_data/config/macros.cfg`) executes `G1 E-8.0 Z0.2 F2700` at the end of every print — an 8 mm retract combined with a 0.2 mm Z lift. This is aggressive for direct drive (normal retractions on this printer are 0.5-1.0 mm) and is specifically intended to prevent post-print ooze on materials with higher melt viscosity. See [`../printer/machine_gcode.md`](../printer/machine_gcode.md) for the full macro.

[^sunlu-batch-consistency]: Users report SUNLU has batch-to-batch colour variation particularly on grey, navy, and pastel shades. The matte variants are affected slightly more than the glossy variants because the filler content varies by colour (more filler = more matte, but also pigment-shifts). Practical recommendation: buy multiple spools of the same colour from the same batch (check lot number on spool label) for large multi-spool projects. <https://forum.bambulab.com/t/sunlu-pla-meta-settings/23934>
