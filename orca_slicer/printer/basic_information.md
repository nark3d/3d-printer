# OrcaSlicer Printer Settings — Basic Information tab

> One file per tab. See [`README.md`](README.md) for the index of all tab documents and shared hardware context.
>
> Working document. For each setting on the Basic Information tab: the OrcaSlicer default (value walked through the inheritance chain), the *researched* value (what the OrcaSlicer wiki / community / Klipper docs recommend), our current value, and a comments field documenting the reasoning behind our choice.
>
> **Important**: many printer settings reflect *actual hardware geometry* — the printable area, the extruder clearance, the nozzle diameter. These have a *correct* answer (what the hardware can physically do), not a "preference" answer. **The most important findings on this tab are the mismatches between our OrcaSlicer values and what Klipper actually does** — those mismatches mean the slicer is planning gcode based on wrong assumptions.

---

## ⚠️ Findings flagged at the top

Three notable issues identified during research:

| # | Setting | OrcaSlicer has | Klipper actually does | Impact |
|---|---|---|---|---|
| 1 | `printable_area` | 260 × 240 mm | 260 × 245 mm (per `printer.cfg` `position_max`) | **5 mm of usable Y is being wasted.** Slicer refuses to place objects in the rear 5 mm. |
| 2 | `machine_max_acceleration_x/y/extruding` | **40 000 mm/s²** | **10 000 mm/s²** (per `printer.cfg` `max_accel`) | **4× mismatch.** Slicer plans gcode assuming 4× the actual accel headroom. Klipper silently caps to 10000, but the slicer's velocity/timing planning was already wrong. |
| 3 | `extruder_clearance_*` | radius 65, height to rod 36, height to lid 140 | (untested for VzBot CNC toolhead) | These come from the generic Klipper base profile, not from measuring the actual toolhead. **Untested for our hardware** — only matters if we use sequential ("by object") printing. |

The acceleration mismatch (#2) is the highest-impact item. Fixing it doesn't change print speed (Klipper enforces 10000 regardless), but it makes the slicer's gcode planning *honest* — the slicer will no longer assume it can decelerate from 600 mm/s to 0 in distances that require 40k accel.

---

## Printer identity

These are the top-of-tab identity fields. They're metadata — no gcode impact, but they determine how OrcaSlicer presents this profile in the UI and which filament/process profiles list it as compatible.

| Setting | OrcaSlicer default | Researched value | Our value | Comments |
|---|---|---|---|---|
| Printer name | (user-set) | — | **`ZeroG Mercury One`** | The profile label shown in the printer dropdown. Matches the hardware. **DECISION: no change** — obvious, matches physical printer. |
| Printer model | `Generic Klipper Printer` (from `MyKlipper 0.4 nozzle` parent) | — | `Generic Klipper Printer` *(inherited)* | Internal model ID used to filter compatible filaments/processes. "Generic Klipper Printer" is the parent profile's model. **DECISION: no change** — changing this would break compatibility links with existing filament/process profiles. |
| Printer variant (nozzle diameter) | `0.4` | `0.4` (matches hardware) | `0.4` *(inherited)* | Nozzle diameter this profile is configured for. Our physical hardware is 0.4 mm hardened steel. **DECISION: no change** — matches hardware exactly. |
| Printer preset ID | (matches name) | — | `ZeroG Mercury One` | Unique identifier for this preset — same as name. **DECISION: no change**. |
| Extruder count | `1` | `1` (single-extruder printer) | `1` *(inherited)* | Number of extruders on the physical printer. We have one Orbiter 2.5. **DECISION: no change** — matches hardware. |
| Extruder variant | (varies) | `Direct Drive Standard` | `Direct Drive Standard` | The generic extruder class. Our Orbiter 2.5 is a direct-drive geared extruder — "Direct Drive Standard" is the correct generic class. **DECISION: no change** — accurate metadata. |
| Print host URL | (empty) | `http://192.168.x.x` (our Moonraker) | `192.168.x.x` | The IP of our Moonraker instance. Used by Orca's "Upload to printer" button. **DECISION: no change** — matches our Klipper host. (The Print Host sub-tab has additional settings like API key — not on the Basic Info tab itself, but related.) |

---

## Printable space

| Setting | OrcaSlicer default | Researched value | Our value | Comments |
|---|---|---|---|---|
| Printable area | 250 × 250 mm (parent profile) | **260 × 245 mm** (matches Klipper `position_max` X=260 Y=245) | **260 × 240 mm** | **Mismatch — Y is 5 mm short of actual.** Klipper allows 245 mm Y, but our Orca profile has 240. **DECISION: CHANGE to `0x0, 260x0, 260x245, 0x245`** to match Klipper `position_max`. Recovers 5 mm of usable Y. |
| Excluded bed area | `0x0` (none) | (none, unless physical obstruction) [^bed-exclude] | `0x0` *(inherited)* | Defines bed regions where the toolhead cannot travel — used for clip-down spots, hardware obstructions, etc. We have no bed obstructions. **DECISION: no change** — default fine. |
| Printable height | 250 mm (parent) | **258 mm** (matches Klipper `[stepper_z] position_max`) | 250 mm *(inherited)* | **Minor mismatch — Z is 8 mm short.** Klipper allows 258 mm but we inherit 250 from the parent. **DECISION: CHANGE to `258`** to match Klipper `position_max`. Recovers 8 mm of vertical envelope. |
| Support multi bed types | `false` | `true` (if you have multiple physical bed plates) | **`true`** | Lets us define multiple bed surface types (textured PEI, smooth PEI, etc.) and pick per slice. Framework is harmless even with one current plate. **DECISION: no change** — keep override ON, future-proof. |
| Best object position | (varies, often 0.5/0.5) | **0.5/0.5 (centre)** for CoreXY [^bed-position] | (default 0.5/0.5) *(inherited)* | Auto-arrange placement preference. 0.5/0.5 = "centre of bed", correct for CoreXY (no first-layer adhesion bias toward a corner). **DECISION: no change** — default is correct. |
| Z offset | 0 mm | **0 mm** (Klipper handles Z offset via `[probe] z_offset` and `Z_OFFSET_APPLY_PROBE` macro) | 0 mm *(inherited)* | Adding a slicer-side Z offset would double-up with Klipper's probe-based Z offset. **DECISION: no change — MUST stay at 0 for Klipper**. |
| Preferred orientation | 0° | (any value works — affects auto-arrange) | (default 0°) | Auto-arrange rotation preference. No quality impact. **DECISION: no change** — default fine. |

---

## Advanced

| Setting | OrcaSlicer default | Researched value | Our value | Comments |
|---|---|---|---|---|
| Printer structure | `Undefined` | **`CoreXY`** [^printer-structure] | `Undefined` *(inherited)* | Tells OrcaSlicer the kinematic style. Options: `CoreXY`, `i3`, `H-bot`, `Delta`, `MarkForged`, `Undefined`. Our hardware is CoreXY (Mercury One.1). **DECISION: CHANGE to `CoreXY`** — accurate metadata, may unlock CoreXY-specific gcode planning optimisations. Low risk. |
| G-code flavor | `klipper` (Klipper base) | **`klipper`** [^gcode-flavor] | `klipper` *(inherited)* | Klipper requires its own gcode dialect for custom commands and macro syntax. **DECISION: no change — MUST stay at `klipper`**. |
| Pellet Modded Printer | `false` | `false` (we use filament) | (default) | Special mode for printers using pellet extruders. **DECISION: no change** — we use filament. |
| Disable set remaining print time | `false` | `false` *(default)* | (default) | If true, suppresses M73 progress commands. Mainsail uses M73 for the progress bar. **DECISION: no change** — leave false so Mainsail progress works. |
| G-code thumbnails | (varies, often 32x32 + 400x300/PNG) | **32x32 + 400x300/PNG** for Mainsail [^thumbnails] | **48x48/PNG, 300x300/PNG** | Embeds preview images in the gcode file. Mainsail's documented recommendation is 32x32 + 400x300 PNG. Our values work but are non-standard. **DECISION: CHANGE to `32x32/PNG, 400x300/PNG`** — matches Mainsail's documented UI sizes exactly. |
| Use relative E distances | `false` (Klipper base) | **`true`** for Klipper [^relative-e] | **`true`** | Klipper *requires* relative E (M83 mode). We override correctly. **DECISION: no change — MUST stay `true` for Klipper**. |
| Use firmware retraction | `false` | `false` (let the slicer handle retraction) | (default `false`) | Firmware retraction (G10/G11) delegates retraction to Klipper. Not recommended — slicer-side retraction gives per-feature control. **DECISION: no change** — leave `false`. |
| Time cost | 0 (money/h) | (cosmetic, optional) | (default 0) | Display value for cost-per-hour estimates. No print impact. **DECISION: no change** — we don't track print cost by time; leave 0. Could be set later to UK electricity rate (~£0.30/kWh × ~0.2 kW = ~£0.06/h) if we ever want cost tracking. |

---

## Cooling Fan

| Setting | OrcaSlicer default | Researched value | Our value | Comments |
|---|---|---|---|---|
| Fan speed-up time | 0 s | **0** for direct-driven fans (no compensation needed); higher for laggy fans [^fan-timing] | (default 0) | Compensation for fans that take time to ramp up. Our Phaetus 4020/5015 fans respond essentially instantly. **DECISION: no change** — default 0 is correct. |
| Fan kick-start time | 0 s | 0 (modern fans don't need this) | (default 0) | Brief full-speed pulse to help laggy fans start at low PWM. Our fans don't need this. **DECISION: no change** — default 0 is correct. |
| Only overhangs | (default `true`) | `true` *(default)* | **`true`** | If kick-start is enabled, only apply it on overhang fan changes. Moot since kick-start is off. **DECISION: no change**. |

---

## Extruder Clearance

| Setting | OrcaSlicer default | Researched value | Our value | Comments |
|---|---|---|---|---|
| Radius | 65 mm | **needs measurement** for VzBot CNC toolhead [^extruder-clearance] | 65 mm *(inherited from generic Klipper base)* | XY distance from nozzle centre to the outermost toolhead component. Used for sequential ("by object") printing collision detection. **Untested for our hardware.** Only matters if we switch `print_sequence` to `by object`. **DECISION: no change (defer)** — we use `by layer` per `others_PLA.md`, so this value is currently unused. Needs physical measurement of the VzBot CNC toolhead before any meaningful update. |
| Height to rod | 36 mm | **needs measurement** [^extruder-clearance] | 36 mm *(inherited)* | Vertical distance from nozzle tip to lower bound of X gantry. Used for sequential printing Z-clearance checks. **Untested.** **DECISION: no change (defer)** — only matters with `by object` printing. |
| Height to lid | 140 mm | **needs measurement** [^extruder-clearance] | 140 mm *(inherited)* | Vertical distance from nozzle tip to upper enclosure ceiling. Limits max Z in sequential printing. **Untested.** **DECISION: no change (defer)** — only matters with `by object` printing. |

> **For us this section doesn't currently matter** because we use `print_sequence: by layer` (per the Process Others tab), so the slicer never plans toolhead-to-object collision avoidance. If we ever switch to `by object`, we'd need to measure these properly first.

---

## Adaptive bed mesh

| Setting | OrcaSlicer default | Researched value | Our value | Comments |
|---|---|---|---|---|
| Bed mesh min | -99999, -99999 (effectively unbounded) | **5, 36** to match Klipper `[bed_mesh] mesh_min` (confirmed from live `printer.cfg`) | -99999, -99999 *(default)* | Minimum X,Y probe point the slicer should suggest for adaptive mesh. Klipper enforces the real bounds regardless, so default `-99999` works. **DECISION: CHANGE to `5, 36`** to match Klipper `mesh_min: 5, 36` exactly. Low risk, makes slicer planning honest. |
| Bed mesh max | 99999, 99999 | **255, 240** to match Klipper `[bed_mesh] mesh_max` (confirmed from live `printer.cfg`) | 99999, 99999 *(default)* | Same as above for the maximum. **DECISION: CHANGE to `255, 240`** to match Klipper `mesh_max: 255, 240`. |
| Probe point distance | 50, 50 | 35-50 mm community typical; ours OK given bed flatness | 50, 50 *(default)* | Spacing between probe points in adaptive mesh. 50 mm spacing on a 260 × 245 bed produces ~5 × 5 = 25 points for a full bed mesh, which is plenty for our Beacon probe. Tighter spacing is overkill unless the bed is warped. **DECISION: no change** — default 50 is fine. |
| Mesh margin | 0 mm | 0 mm *(default)* | 0 mm *(default)* | Extra margin around the print bounding box for adaptive mesh. 0 = mesh exactly the print area. **DECISION: no change** — default fine. |

> **Adaptive mesh integration**: this section is for the "adaptive" component of bed meshing (only mesh the area the print actually uses). For it to work, both the slicer (here) and Klipper (`BED_MESH_CALIBRATE ADAPTIVE=1` in our `PRINT_START` macro) need to be configured. Klipper side is already correct.

---

## Accessory

| Setting | OrcaSlicer default | Researched value | Our value | Comments |
|---|---|---|---|---|
| Nozzle type | `undefine` (Klipper base) | **`hardened_steel`** for our Phaetus Rapido 2 HF [^nozzle-type] | **`hardened_steel`** | Phaetus Rapido 2 HF ships with hardened steel. Setting correctly lets Orca display the right icon and warn about incompatible abrasive filaments. **DECISION: no change** — override is correct, matches hardware. |

---

## Summary of recommended Basic Information tab changes

### Strongly recommended (fixes real mismatches with Klipper)

| Setting | Currently | Recommend | Reason |
|---|---|---|---|
| `printable_area` | 260 × 240 | **260 × 245** | Matches Klipper `position_max` Y=245. Recovers 5 mm of usable bed area |
| `printable_height` | 250 | **258** | Matches Klipper Z `position_max`. Recovers 8 mm of vertical print envelope |
| `printer_structure` | `Undefined` | **`CoreXY`** | Accurate metadata. May unlock CoreXY-specific gcode planning |
| `machine_max_acceleration_*` (in Motion ability tab — flagged here for reference) | 40 000 / 20 000 | **10 000 / 5 000** to match Klipper | Slicer planning is currently based on 4× the actual ceiling — incorrect velocity/timing decisions in tight features. **Will be covered in detail in `motion_ability.md`** |

### Optional refinements

| Setting | Currently | Could refine to | Reason |
|---|---|---|---|
| `g_code_thumbnails` | `48x48/PNG, 300x300/PNG` | `32x32/PNG, 400x300/PNG` | Mainsail's documented recommended sizes. Both work, but the documented values match Mainsail's UI exactly |
| `bed_mesh_min` | -99999, -99999 | 5, 36 | Match Klipper `[bed_mesh] mesh_min`. Cosmetic — slicer-side constraint, Klipper enforces the real limits regardless |
| `bed_mesh_max` | 99999, 99999 | 255, 240 | Match Klipper `[bed_mesh] mesh_max`. Same comment |

### Confirmed correct (no change needed)

- `support_multi_bed_types: true` — useful framework even with one current plate
- `best_object_position: 0.5/0.5` — correct for CoreXY centre-print
- `z_offset: 0` — required for Klipper (Klipper handles Z offset itself)
- `gcode_flavor: klipper` — required
- `use_relative_e_distances: true` — required for Klipper
- `use_firmware_retraction: false` — slicer-side retraction is preferred for Klipper
- `nozzle_type: hardened_steel` — correct for our Rapido 2 HF
- `extruder_clearance_*` — inherited generic Klipper values; **untested but only matter for sequential printing which we don't use**

### Not actionable until we measure

- `extruder_clearance_radius`, `_height_to_rod`, `_height_to_lid` — need physical measurement of the VzBot CNC toolhead. Only matters if we ever switch to `print_sequence: by object`. **Defer until needed.**

---

## References

[^bed-exclude]: OrcaSlicer Excluded Bed Area — defines bed regions where the toolhead cannot travel. Used for clip-down points, fixed hardware obstructions, etc. <https://github.com/SoftFever/OrcaSlicer/wiki/printer_basic_information_printable_area>

[^bed-position]: OrcaSlicer Best Object Position — auto-arrange placement preference. 0.5/0.5 means "centre of bed", which is the right choice for any printer where there's no first-layer adhesion bias (true for our textured PEI + Beacon-tuned Z offset).

[^printer-structure]: OrcaSlicer Printer Structure — describes the kinematic style. Options: CoreXY, i3, H-bot, Delta, MarkForged, Undefined. Setting this correctly is metadata that may unlock kinematic-specific gcode planning optimizations. <https://github.com/SoftFever/OrcaSlicer/wiki/printer_basic_information_advanced>

[^gcode-flavor]: OrcaSlicer G-code Flavor — Klipper requires the `klipper` gcode flavor for its custom commands and macro syntax. <https://github.com/SoftFever/OrcaSlicer/wiki/printer_basic_information_advanced> · <https://www.klipper3d.org/Slicers.html>

[^thumbnails]: Mainsail G-code Thumbnails documentation — recommended sizes are **32x32 + 400x300 PNG** for full Mainsail UI compatibility. The 32x32 is shown in the file list, the 400x300 is shown on the print page. Other sizes work but may not display optimally. <https://docs.mainsail.xyz/features/thumbnails/> · <https://docs.mainsail.xyz/slicers/orcaslicer/>

[^relative-e]: Klipper requires `relative_e_distances: true` (M83 mode). Without it, Klipper interprets E values incorrectly and produces wrong extrusion. This is one of the canonical Klipper slicer settings. <https://www.klipper3d.org/Slicers.html>

[^fan-timing]: Fan timing settings (speed-up time, kick-start time) compensate for laggy or hard-to-start fans. Modern 4020/5015/5020 fans (which our Phaetus toolhead uses) start cleanly at low PWM and don't need either compensation. Default 0 is correct.

[^extruder-clearance]: OrcaSlicer Extruder Clearance Wiki — three values describing toolhead geometry for sequential ("by object") printing collision avoidance. **Radius**: distance from nozzle to the outermost toolhead component in XY (BBL X1C example: 68 mm because of the LIDAR module). **Height to rod**: vertical distance to the lower bound of any X gantry component. **Height to lid**: vertical distance to the upper enclosure ceiling. **Untested for VzBot CNC toolhead** — would need physical measurement if we ever use sequential printing. Generic Klipper base values (65 / 36 / 140) are inherited. <https://github.com/SoftFever/OrcaSlicer/wiki/printer_basic_information_extruder_clearance>

[^adaptive-mesh]: Adaptive bed meshing requires both the slicer (this tab's bed mesh settings) and the firmware (Klipper's `[bed_mesh]` config + `BED_MESH_CALIBRATE ADAPTIVE=1` in `PRINT_START`) to be configured. The slicer-side `bed_mesh_min/max` are *constraints* — Klipper enforces the real bounds regardless. Probe point distance affects mesh density.

[^nozzle-type]: OrcaSlicer Nozzle Type — declares the nozzle material so Orca can warn about incompatible filaments (e.g. abrasive CF/GF filaments on brass nozzles). Phaetus Rapido 2 HF ships with a hardened steel nozzle. Setting this correctly is good practice but mostly cosmetic for non-abrasive filaments like PLA.
