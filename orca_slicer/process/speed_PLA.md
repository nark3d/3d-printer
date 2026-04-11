# OrcaSlicer Speed tab settings — PLA quality profile

> One file per tab. See [`README.md`](README.md) for the index of all tab documents and shared hardware context.
>
> **Speed tab is the most data-intensive of the process tabs.** This is where the OrcaSlicer-bundled VzBot profile values are *most directly relevant* as a reference, and also where the "VzBot toolhead vs VzBot printer" caveat *matters most*. Our heavier CNC toolhead means VzBot bundled speeds and accelerations are an **upper bound**, not a target.
>
> **Hardware-specific reasoning anchor**: input shaper measured at MZV X=61 Hz / Y=42 Hz with the LIS2DW. For comparison, a typical lightweight Voron / VzBot Vz-Printhead has Y resonance around 50-65 Hz. Lower resonance frequency = lower acceleration headroom before ringing becomes visible. Klipper's formula: ringing frequency ≈ V × N / D, so for a given input shaper frequency the safe outer-wall *velocity* scales with the resonance frequency. We have ~70 % of a typical Voron's headroom on Y.
>
> Working document. Same column structure as the other tabs.

---

## Speed tab

### Initial layer speed

| Setting | OrcaSlicer default | Researched value | Our value | Comments |
|---|---|---|---|---|
| Initial layer | 45 mm/s (base) / 50 mm/s (Klipper) | 50-100 mm/s [^vzbot-bundled] | (default 50) *(inherited)* | Conservative first-layer speed for reliable adhesion. **DECISION: no change** — 50 is safe for PLA. |
| Initial layer infill | 45 mm/s (base) / 105 mm/s (Klipper) | 50-120 mm/s [^vzbot-bundled] | **60 mm/s** | Override 60 is conservative — reliable adhesion on tall first-layer features. **DECISION: no change** — override is quality-first. |
| Initial layer travel speed | (inherits travel_speed) | (inherits) | (inherits) | Inherits from travel_speed. **DECISION: no change** — inherit is fine. |

### Other layer speed

| Setting | OrcaSlicer default | Researched value | Our value | Comments |
|---|---|---|---|---|
| Outer wall | 45 mm/s (base) / 120 mm/s (Klipper) | **100-150 mm/s** for our heavy toolhead [^vzbot-bundled] [^outer-wall-speed] | **100 mm/s** | Override 100 is the quality-first choice. Could go to 120 without quality loss, but slower outer wall = better surface finish. **DECISION: no change** — override is correct for quality goal. |
| Inner wall | 80 mm/s (base) / 200 mm/s (Klipper) | **180-220 mm/s** [^vzbot-bundled] [^outer-wall-speed] | **180 mm/s** | Inner walls are hidden. Klipper default is 200. No quality cost to going faster. **DECISION: CHANGE to `200`** — matches Klipper default, saves time per print with no quality risk. |
| Small perimeter | 50% (relative to outer wall) | 50% *(default)* | (default) | Small features get 50% of outer wall speed to avoid ringing in tight curves. **DECISION: no change** — default is well-tuned. |
| Sparse infill | 150 mm/s (base) / 200 mm/s (Klipper) | 180-250 mm/s [^vzbot-bundled] | (default 200) *(inherited)* | Hidden, benefits most from speed. Rapido 2 HF practical max ~22-28 mm³/s. **DECISION: no change** — default 200 is fine; speed tuning can wait. |
| Internal solid infill | 150 mm/s (base) / 200 mm/s (Klipper) | 180-250 mm/s [^vzbot-bundled] | (default 200) *(inherited)* | Same flow constraint as sparse infill. **DECISION: no change** — default 200 is fine. |
| Top surface | 50 mm/s (base) / 100 mm/s (Klipper) | 100-150 mm/s [^vzbot-bundled] | **80 mm/s** | Very conservative. With ironing re-flattening the top, can go faster without quality loss. Klipper default is 100. **DECISION: CHANGE to `100`** — matches Klipper default, acceptable with ironing enabled. |
| Gap infill | 30 mm/s (base) / 100 mm/s (Klipper) | 60-100 mm/s | **60 mm/s** | Slower gap fill = cleaner deposition of tiny moves. Override 60 is a sensible quality choice. **DECISION: no change** — override is correct. |
| Support | 150 mm/s (base) | 150 mm/s [^vzbot-bundled] | (default 150) *(inherited)* | Support doesn't need quality. **DECISION: no change** — default is correct. |
| Support interface | 80 mm/s (base) | 80 mm/s [^vzbot-bundled] | (default 80) *(inherited)* | Support interface touches the print, needs slightly more care. **DECISION: no change** — default is correct. |
| Bridge | 50 mm/s (base) | **50-80 mm/s** [^bridging-wiki] | (default 50) *(inherited)* | Bridge speed pairs with bridge flow ratio (1.0 flow → 50 mm/s). Pairs correctly with the recommended 0.95 bridge flow from Quality tab. **DECISION: no change** — default 50 pairs correctly with our bridge flow. |
| Internal bridge | 150 mm/s (base) / 100% (relative, VzBot) | 100-150 mm/s | (default) | Internal bridges are inside the print, don't need surface quality, just need not to lump. **DECISION: no change** — default is correct. |
| Ironing | 30 mm/s | 30-40 mm/s [^ironing-wiki] | **40 mm/s** | Already reviewed in ironing deep dive. **DECISION: deferred to ironing review** — the user asked to review ironing separately. |

### Overhang speed

| Setting | OrcaSlicer default | Researched value | Our value | Comments |
|---|---|---|---|---|
| Slow down for overhang | `true` | `true` *(quality necessity)* | (default) | Master switch for per-overhang slowdown. **DECISION: no change** — default on is correct. |
| Enable overhang speed | `true` | `true` *(default)* | (default) | Enables the per-overhang slowdown values below. **DECISION: no change** — default on is correct. |
| 0-25% overhang speed | 0 (auto) | 0 (auto) [^vzbot-bundled] | (default) | 0 = "no slowdown" — <25% isn't really an overhang. **DECISION: no change** — default is correct. |
| 25-50% overhang speed | 50 mm/s (base) / 120 mm/s (VzBot) | **80-120 mm/s** [^vzbot-bundled] | (default 50) | Default 50 is conservative. **DECISION: no change** — quality-first, slower is better on moderate overhangs. |
| 50-75% overhang speed | 30 mm/s (base) / 80 mm/s (VzBot) | **50-80 mm/s** [^vzbot-bundled] | (default 30) | Default 30 is conservative — good for quality. **DECISION: no change**. |
| 75-100% overhang speed | 10 mm/s (base) / 30 mm/s (VzBot) | **15-30 mm/s** [^vzbot-bundled] | (default 10) | Default 10 is very conservative — safest for extreme overhangs. **DECISION: no change** — quality-first. |
| Bridge overhang | (varies) | (default) | (default) | Treat detected bridges as overhangs for speed. **DECISION: no change** — default is correct. |

**Note on overhang speeds**: VzBot bundled values (120/80/30) are aggressive — they reflect the VzBot devs' lightweight head + good cooling. Our defaults (50/30/10) are 2-3× more conservative, which is correct for our heavier toolhead and is the *quality-first* choice. **No change recommended** — accept slower overhangs for cleaner results.

### Travel speed

| Setting | OrcaSlicer default | Researched value | Our value | Comments |
|---|---|---|---|---|
| Travel | 200 mm/s (base) / 350 mm/s (Klipper) | **600-800 mm/s** [^vzbot-bundled] | **500 mm/s** | Our printer's `max_velocity: 600`. Anything above 600 clamps at 600. **DECISION: CHANGE to `600`** — matches the printer's ceiling exactly, saves travel time with no downside. |
| Z hop speed | (printer config) | (printer config) | (printer config) | Set in the printer profile. **DECISION: N/A** — not a process-tab setting. |

### Acceleration

| Setting | OrcaSlicer default | Researched value | Our value | Comments |
|---|---|---|---|---|
| Default acceleration | 1000 mm/s² (base) / 5000 (Klipper) | **5000-8000** for our heavy toolhead [^vzbot-bundled] [^accel-input-shaper] | **5500 mm/s²** | Override 5500 is conservative, on par with Klipper default. Pushing higher needs print testing for ringing. **DECISION: no change** — empirically set, leave alone. |
| Outer wall acceleration | 700 mm/s² (base) | **3000-5000** [^vzbot-bundled] [^accel-input-shaper] | (default 5000?) *(inherited)* | Lower outer wall accel reduces ringing. Optional refinement to set explicitly. **DECISION: no change** — inherit from parent; explicit override is a speed tuning exercise that needs ringing test first. |
| Inner wall acceleration | 1000 mm/s² (base) / 5000 (Klipper) | **5000-7000** [^vzbot-bundled] [^accel-input-shaper] | **4000 mm/s²** | Inner walls are hidden, could safely push higher. **DECISION: no change** — empirically set, leave alone. Push only after ringing test. |
| Top surface acceleration | 1000 mm/s² (base) / 3000 (Klipper) | **3000-5000** [^vzbot-bundled] | (default 3000) | Top surface is visible — lower accel helps quality. **DECISION: no change** — default 3000 is correct. |
| Travel acceleration | 1000 mm/s² (base) / 7000 (Klipper) | **10 000-20 000** [^vzbot-bundled] | **10 000 mm/s²** | Matches our Klipper `max_accel` ceiling. Anything higher clamps. **DECISION: no change** — override is exactly right. |
| Initial layer acceleration | 500 mm/s² (base and Klipper) | 500-1000 [^vzbot-bundled] | (default 500) *(inherited)* | First layer needs gentle accel for adhesion. **DECISION: no change** — default 500 is correct. |
| Sparse infill acceleration | (newer Orca, varies) | varies | (default) | Newer Orca splits this from default acceleration. **DECISION: no change** — default works. |
| Internal solid infill acceleration | (newer Orca) | varies | (default) | Same. **DECISION: no change** — default works. |

### Jerk (XY)

> **Klipper note**: Klipper uses `square_corner_velocity` (currently 5.0 mm/s) instead of jerk. The OrcaSlicer jerk settings are for Marlin firmware and are **mostly ignored by Klipper**. Klipper firmware emits its own corner velocity behaviour. Setting jerk values in the slicer won't break anything but won't have effect either.

| Setting | OrcaSlicer default | Researched value | Our value | Comments |
|---|---|---|---|---|
| Default jerk | (varies) | N/A for Klipper | (default) | Marlin-only. Klipper ignores M205. **DECISION: no change** — N/A. |
| Outer wall jerk | (varies) | N/A for Klipper | (default) | Marlin-only. **DECISION: no change** — N/A. |
| Inner wall jerk | (varies) | N/A for Klipper | (default) | Marlin-only. **DECISION: no change** — N/A. |
| Travel jerk | (varies) | N/A for Klipper | (default) | Marlin-only. **DECISION: no change** — N/A. |
| First layer jerk | (varies) | N/A for Klipper | (default) | Marlin-only. **DECISION: no change** — N/A. |

### Pressure equalizer / Extrusion rate smoothing (ERS)

| Setting | OrcaSlicer default | Researched value | Our value | Comments |
|---|---|---|---|---|
| Max volumetric extrusion rate slope | 0 (off) | **100-150 mm³/s²** for Voron-class printers [^pressure-equalizer] | **300 mm³/s²** | Voron community uses ~100, Bambu 200-400. Our 300 is Bambu-range — aggressive. **DECISION: no change for now** — ERS interacts with per-filament pressure advance, and changing it without a calibration print is just replacing one untested value with another. Revisit during per-filament tuning. |
| Max volumetric extrusion rate slope segment length | 1 mm | 1 mm [^pressure-equalizer] | **1 mm** | Segment length over which the slope is smoothed. Community-recommended value. **DECISION: no change** — override matches community standard. |

> **Klipper-specific advice for pressure equalizer**: If you use a direct-drive extruder (which we do), the OrcaSlicer wiki recommends reducing your Klipper `pressure_advance` `smooth_time` to 0.02 *before* tuning the extrusion rate slope. This gives lower deviations and more accurate slope behaviour. Worth checking what your current Klipper pressure_advance smooth_time is — set in the filament profile or `[extruder]` section.

---

## Summary of recommended Speed tab changes from current overrides

The Speed tab analysis identifies these settings:

### Recommended changes (small refinements, not blocking)

| Setting | Currently | Could refine to | Reason |
|---|---|---|---|
| `inner_wall_speed` | 180 mm/s | **200 mm/s** | Klipper default is 200. Our 180 is conservative. Inner walls are hidden, no quality cost to going faster. Saves a few minutes per print. |
| `top_surface_speed` | 80 mm/s | **100-120 mm/s** | Klipper default is 100, VzBot bundled is 200. Our 80 is very conservative. With ironing enabled, the top surface is re-flattened anyway. Safe to bump. |
| `travel_speed` | 500 mm/s | **600 mm/s** | Matches `max_velocity` ceiling. Anything above 600 would clamp anyway. Saves travel time. |
| `outer_wall_acceleration` | (inherited, probably 5000) | **3000-4000** *(explicit)* | Outer wall surface quality benefits from slightly lower accel. VzBot bundled uses 5000; Voron community often uses 3000-4000 explicitly for outer walls. Worth setting explicitly rather than inheriting. |
| `max_volumetric_extrusion_rate_slope` | 300 | **100-150** | Voron community uses ~100 with 1 segment. Our 300 is closer to Bambu's range — too aggressive for our setup if we have well-tuned pressure advance. |

### Optional speed-up refinements (if you want faster prints, not strictly quality)

| Setting | Currently | Could push to | Reason |
|---|---|---|---|
| `default_acceleration` | 5500 | 6000-7000 | Voron community runs 7000-12000 after input shaping; we have less headroom but 6000-7000 is realistic |
| `inner_wall_acceleration` | 4000 | 5000-6000 | Inner walls are hidden, can take more accel without quality loss |
| `sparse_infill_speed` | (default 200) | 220-250 | Rapido 2 HF can handle the volumetric flow at 0.45 mm × 0.2 mm × 220-250 mm/s = 19.8-22.5 mm³/s — within practical Rapido 2 HF range |

### Confirmed correct (no change needed)

- `outer_wall_speed: 100` — quality choice, defensible. Could go to 120 if you want
- `gap_infill_speed: 60` — quality choice
- `initial_layer_infill_speed: 60` — quality choice
- `ironing_speed: 40` — within safe range, slightly faster than wiki default but fine
- `travel_acceleration: 10000` — matches `max_accel` ceiling, can't usefully go higher without raising `max_accel` first
- All overhang speeds (default values) — quality-first, slower than VzBot bundled but appropriate for our heavier toolhead
- `bridge_speed: 50` — pairs correctly with the recommended `bridge_flow: 0.95` from the Quality tab
- All Marlin jerk settings — ignored by Klipper, no change needed

### What needs hardware testing

**Acceleration values are inherently empirical.** The "right" max accel depends on whether your prints actually show ringing artefacts at the current value. Process for tuning:

1. Print a test cube at the current accel
2. Inspect outer walls for ringing (faint vertical lines after sharp corners)
3. If clean → could push higher
4. If ringing → reduce until clean

This is a "tune until print quality breaks, back off" loop. The values in this document are starting points based on community data, not absolute optimums for your specific printer.

---

## References

[^vzbot-bundled]: OrcaSlicer-bundled VzBot process profile (`fdm_process_Vzbot_common.json`). Maintained by VzBot devs and shipped with OrcaSlicer. **Critical caveat for the Speed tab**: calibrated for the *lightweight* Vz-Printhead on the VzBot 330 AWD frame. Our heavier CNC toolhead variant has lower input shaper resonance frequency (~70 % of typical Voron) and therefore lower acceleration headroom. Speed/acceleration values from this source should be treated as an **upper bound**, not a target. <https://github.com/SoftFever/OrcaSlicer/tree/main/resources/profiles/Vzbot/process>

[^outer-wall-speed]: Community consensus from Klipper / Voron forums on input-shaper-tuned printers: outer wall speeds in the **100-200 mm/s** range are realistic for PLA on CoreXY printers with input shaping calibrated, with the higher end requiring lighter toolheads and well-tuned input shaping. Quality-first prints typically use the lower end (100-150 mm/s). <https://klipper.discourse.group/t/what-is-the-proper-procedure-to-tune-the-speed-and-acceleration-after-input-shaping/23150>

[^accel-input-shaper]: Voron / Klipper community guidance on accelerations after input shaping: typical range **7000-12000 mm/s²** post-shaping (up from 2000-3000 stock). One user's tuning landed at 6500 mm/s² for clean Benchies including walls. **Caveat**: at very high accelerations, input shaping causes too much smoothing/rounding of outer walls — `max_accel` should be chosen to prevent that visible smoothing. Best practice: take input-shaper-recommended values from `SHAPER_CALIBRATE` and subtract a small margin. Our LIS2DW-derived shaper at 42 Hz Y is on the lower end (heavier toolhead), so our headroom is correspondingly lower than typical Voron numbers — 5000-7000 is realistic for us. <https://klipper.discourse.group/t/what-is-the-proper-procedure-to-tune-the-speed-and-acceleration-after-input-shaping/23150> · <https://forum.vorondesign.com/threads/accelerations-higher-than-input-shaper-suggested.921/>

[^bridging-wiki]: OrcaSlicer Bridging Wiki — bridge speed should be set to roughly 50 % of standard print speed for clean unsupported spans, paired with reduced bridge flow ratio. Recommended pairing: 1.0 flow at 50 mm/s, 0.9 flow at 40 mm/s, 0.8 at 25 mm/s, 0.7 at 15 mm/s. <https://github.com/SoftFever/OrcaSlicer/wiki/quality_settings_bridging>

[^ironing-wiki]: OrcaSlicer Ironing Wiki — ironing speed default is 30 mm/s, with the slow speed being what allows nozzle drag to flatten the top surface. Going much faster (50+) defeats the purpose. <https://github.com/SoftFever/OrcaSlicer/wiki/quality_settings_ironing>

[^pressure-equalizer]: OrcaSlicer Extrusion Rate Smoothing (ERS) Wiki and community guidance. ERS smooths sudden volumetric flow changes between feature transitions. **Voron 2.4 with input shaping**: ~100 mm³/s² with 1 segment is the community sweet spot. **Bambu Lab printers**: higher values (200-400) compensate for less aggressive pressure advance tuning. **Direct drive extruder users**: reduce Klipper `pressure_advance` `smooth_time` to 0.02 *before* tuning ERS for cleaner deviations. <https://github.com/SoftFever/OrcaSlicer/wiki/extrusion-rate-smoothing> · <https://github.com/SoftFever/OrcaSlicer/pull/4264>

[^rapido2-flow]: See [`quality_PLA.md`](quality_PLA.md) `[^rapido2-flow]` — Rapido 2 HF practical PLA volumetric flow is 22-28 mm³/s based on Voron community testing. Manufacturer claim is 45 mm³/s; that's optimistic.
