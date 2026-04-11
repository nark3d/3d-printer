# OrcaSlicer Printer Settings — Machine G-code tab

> One file per tab. See [`README.md`](README.md) for the index of all tab documents and shared hardware context.
>
> The Machine G-code tab contains the slicer-side *bridge* between OrcaSlicer and the Klipper macros in `~/printer_data/config/macros.cfg`. Most of the actual print logic (homing, Z tilt, bed mesh, prime line, end move, etc.) lives in the Klipper `PRINT_START` and `PRINT_END` macros — the slicer just invokes them with the right parameters.
>
> **This is a relatively low-noise tab.** The slicer-side gcode is small and mostly just calls into Klipper. The value of this document is (a) checking the bridge is wired correctly, (b) catching any Marlin-style crud that leaked in from the base profile, and (c) documenting the one override we have (`before_layer_change_gcode`).

---

## ⚠️ Findings flagged at the top

| Item | Status | Note |
|---|---|---|
| Start G-code override is **correct and beneficial** | ✅ | Our custom start gcode parallelises heating with homing and bed mesh — faster than the inherited Marlin-style start that blocks on M190/M109 first |
| End G-code is `PRINT_END` (inherited from Klipper base) | ✅ | Matches our `macros.cfg` which defines a `[gcode_macro PRINT_END]` |
| `before_layer_change_gcode` drops `G92 E0` | ℹ️ | Our override removes the base's `G92 E0`. Defensible in relative-E mode (M83) which is what `PRINT_START` sets. Low-impact either way |
| `machine_pause_gcode` is `PAUSE` (Klipper) | ✅ | Maps to Klipper's built-in `PAUSE` command |
| `time_lapse_gcode` is empty | ℹ️ | `timelapse.cfg` **is** installed on the printer (moonraker-timelapse) but the slicer doesn't emit the `TIMELAPSE_TAKE_FRAME` trigger. If you want timelapses, see the Optional refinements section |
| `change_filament_gcode` is empty | ✅ | Not applicable for single-extruder / no multi-material |

No critical findings. This tab is **in good shape**.

---

## Start G-code

Our value (override in `ZeroG Mercury One.json`):

```
PRINT_START EXTRUDER_TEMP=[nozzle_temperature_initial_layer] BED_TEMP=[bed_temperature_initial_layer_single]
```

OrcaSlicer default (`fdm_klipper_common.json`):

```
M190 S[bed_temperature_initial_layer_single]
M109 S[nozzle_temperature_initial_layer]
PRINT_START EXTRUDER=[nozzle_temperature_initial_layer] BED=[bed_temperature_initial_layer_single]
```

### Why our override is better

The **inherited** version does this, in order:

1. `M190 S80` — set bed temp AND **block until bed reaches 80°C** (~2-4 minutes)
2. `M109 S210` — set hotend temp AND **block until hotend reaches 210°C** (~30-60 seconds more)
3. `PRINT_START …` — now run homing, Z tilt, bed mesh, prime line

Total preheat-then-prepare time: bed wait + hotend wait + homing + Z tilt + mesh.

Our **override** just calls `PRINT_START` with temperatures as parameters. The macro definition (in `~/printer_data/config/macros.cfg`) then does:

```klipper
PRINT_START:
    FANS_ON
    M140 S{BED_TEMP}      # Start heating bed — NON-BLOCKING
    M104 S{EXTRUDER_TEMP} # Start heating extruder — NON-BLOCKING
    leds_bed_heating
    G90 / M83
    G28                    # Home (while heating)
    Z_TILT_ADJUST          # Z motor tilt (while heating)
    BED_MESH_CALIBRATE ADAPTIVE=1  # Bed mesh (while heating)
    M190 S{BED_TEMP}      # NOW block on bed temp
    M109 S{EXTRUDER_TEMP} # NOW block on hotend temp
    PRIME_LINE
```

Total time: max(heat time, home + Z tilt + mesh time) — the slow parts run in **parallel**.

On a cold start, this saves roughly 1-3 minutes. The savings are less on a warm printer but non-zero.

### Parameter naming gotcha

Note the parameter names differ between the two versions:

| Where | Parameter name |
|---|---|
| OrcaSlicer default | `EXTRUDER=...`, `BED=...` |
| Our override       | `EXTRUDER_TEMP=...`, `BED_TEMP=...` |
| `macros.cfg` expects | `params.EXTRUDER_TEMP`, `params.BED_TEMP` |

The override uses the same names the Klipper macro expects. **If you ever revert to the inherited start gcode, the macro would fall back to defaults (BED_TEMP=80, EXTRUDER_TEMP=210) regardless of what OrcaSlicer passes** — because the param names wouldn't match. This is a quiet footgun to be aware of.

### Alternative: use KAMP for prime line

**Currently not used** but worth flagging: `~/printer_data/config/KAMP/Line_Purge.cfg` is installed on the printer. KAMP's `LINE_PURGE` macro draws a prime line that's **adjacent to the first object** instead of at a hardcoded X10 Y10-Y200 position. This means:

- Less wasted filament on prints in the centre/top of the bed
- No collision risk if a print happens to occupy the hardcoded PRIME_LINE area
- Prime line is close to where printing starts → shorter travel → less ooze before the first feature

To use KAMP's LINE_PURGE instead of the hardcoded PRIME_LINE, the macros.cfg `PRINT_START` would need its last line changed from `PRIME_LINE` to `LINE_PURGE`. **This is a Klipper-side change, not a slicer-side change.** Flagging here for visibility.

---

## End G-code

Our value: (inherited) `PRINT_END`

OrcaSlicer default (`fdm_klipper_common`): `PRINT_END`

`fdm_machine_common` base value (shown here only so you know what the inheritance chain *would* give us without the Klipper override):

```
M400 ; wait for buffer to clear
G92 E0 ; zero the extruder
G1 E-4.0 F3600; retract
G91
G1 Z3;
M104 S0 ; turn off hotend
M140 S0 ; turn off bed
M106 S0 ; turn off fan
G90
G0 X110 Y200 F3600
print_end
```

The base is Marlin-style and embeds the retract-lift-park sequence directly. `fdm_klipper_common` overrides this to just `PRINT_END` — delegating everything to the `[gcode_macro PRINT_END]` in macros.cfg. **This is the correct approach** — all the print-end logic lives in one place (Klipper) rather than being split across slicer and firmware.

Our `macros.cfg` `PRINT_END` does:

```klipper
PRINT_END:
    G91                       # Relative positioning
    G1 E-8.0 Z0.2 F2700       # Retract 8mm while lifting Z 0.2mm
    G1 X5 Y5 F3000            # Small XY nudge to drop any stringing
    G1 Z10 F1500              # Lift Z 10mm more
    G90                       # Absolute positioning
    G1 X260 Y240              # Park at back-right corner (260x240 = max X/Y)
    M104 S0                   # Hotend off
    M140 S0                   # Bed off
    M84 X Y E                 # Disable XY and E motors (leaves Z enabled)
    SET_HEATER_TEMPERATURE HEATER=chamber TARGET=0  # Chamber off
    leds_status_ready
```

**Notes on this macro:**

- **8mm retract is aggressive** for direct drive. The Orbiter 2.5 normally retracts 1.0 mm. The 8 mm is a "strong retract to prevent any further ooze once printing ends" — for PLA this is fine, for flexible materials it might cause blob-on-tip issues. Outside scope of the slicer though.
- **Park position is `X260 Y240`** — the corner of our confirmed printable area. This is correct and matches the `printable_area` we recommended setting in [`basic_information.md`](basic_information.md).
- **`M84 X Y E`** leaves the Z motors enabled. Good — prevents Z drop from the bed springs if a print ends and you come back later to inspect.
- **Commented out `M107 S0`** — the part cooling fan is not explicitly turned off. In practice the slicer's final gcode turns it off, and Klipper resets fans on idle anyway.

No slicer-side changes needed.

---

## Before layer change G-code

Our value (override):

```
;BEFORE_LAYER_CHANGE
;[layer_z]
```

OrcaSlicer default (`fdm_klipper_common`):

```
;BEFORE_LAYER_CHANGE
;[layer_z]
G92 E0
```

**The difference is the trailing `G92 E0`** — "reset the extruder position counter to zero". Our override drops it.

### Is dropping G92 E0 safe?

**Yes, in our setup.** The reasoning:

1. `G92 E0` resets the extruder's internal E position counter. This matters in **absolute extrusion mode** (M82), where the slicer tracks a single growing E value and resets it periodically to prevent numerical precision issues (FLOAT32 rounding after tens of thousands of moves).
2. Our `PRINT_START` macro explicitly sets **M83** (relative extrusion). Each `G1 E0.4` move is interpreted as "extrude 0.4mm more from wherever the counter is".
3. In M83 mode, the counter still exists internally but the slicer never references it for positioning decisions. `G92 E0` becomes essentially a no-op for print behaviour.
4. **Klipper uses doubles (FLOAT64) internally** for position tracking, so the precision concern that motivates G92 E0 in Marlin doesn't apply even if you were in M82.

So removing `G92 E0` has **zero practical effect** for us. But also — there's no practical *reason* to remove it. It's a cosmetic preference (cleaner gcode, one fewer command per layer change).

### Recommendation

**Leave the override as-is** (without G92 E0). It's harmless. But also **leave a mental bookmark**: if you ever enable absolute extrusion mode in the slicer (`use_relative_e_distances: 0`), you MUST put G92 E0 back, because then the counter does grow unbounded and the firmware will eventually overflow / lose precision.

OrcaSlicer's default is relative E, and we don't have a reason to change that.

---

## Layer change G-code

Our value: (inherited) `;AFTER_LAYER_CHANGE\n;[layer_z]`

OrcaSlicer default (`fdm_klipper_common`): `;AFTER_LAYER_CHANGE\n;[layer_z]`

Both comments. No actual gcode emitted — these are markers so the gcode viewer and Klipper Estimator can identify layer boundaries. **No changes needed.**

### Why comments instead of actual gcode?

The OrcaSlicer-standard pattern is:

- `before_layer_change_gcode`: runs **just before** the layer change move (end of current layer)
- `layer_change_gcode`: runs **just after** the layer change move (start of new layer)

If you wanted to emit e.g. a per-layer relay toggle, beep, timelapse frame, or custom feature, you'd add it to one of these. Empty comments are the "no custom action" default.

---

## Time lapse G-code

Our value: (empty — not set)

OrcaSlicer default: (empty)

### Enabling timelapse

`timelapse.cfg` is installed on the printer (from `moonraker-timelapse`, included in our MainsailOS image). The way it works:

1. Moonraker-timelapse defines a `[gcode_macro TIMELAPSE_TAKE_FRAME]` in `timelapse.cfg`
2. When the slicer emits that macro call on layer change, the camera takes a frame
3. At print end, Moonraker stitches the frames into an MP4

To enable, set the Time Lapse G-code in OrcaSlicer to:

```
TIMELAPSE_TAKE_FRAME
```

**This is optional** and not needed for normal printing. If you never use timelapses, leave empty.

---

## Pause G-code

Our value: (inherited) `PAUSE`

OrcaSlicer default (`fdm_klipper_common`): `PAUSE`

`fdm_machine_common` base: `M601` (Marlin-style)

The Klipper override is correct — calls Klipper's built-in `PAUSE` command (which is also extended by the `[gcode_macro RESUME]` in our `macros.cfg` to restore temperatures). **No changes needed.**

### Related: filament change via M600

`macros.cfg` defines a `[gcode_macro M600]` that:

1. Saves gcode state
2. Calls `PAUSE`
3. Holds temperatures
4. Lifts Z 10mm, retracts 3mm
5. Moves to X130 Y0 (front of bed)
6. Prints a message prompting manual filament change

This is manually invokable from the console (e.g. to do a filament swap at a specific layer). It's not in the slicer's change_filament_gcode because we don't do multi-material swaps — but it's there if you need it.

---

## Change filament G-code

Our value: (inherited) empty string `""`

OrcaSlicer default (`fdm_klipper_common`): empty string `""`

Only relevant for multi-material setups where the slicer emits a colour change mid-print. **N/A for us** — we don't do multi-material.

---

## Template custom G-code

Our value: (not set)

Used as a placeholder for a shared gcode snippet that can be referenced from other fields via `[template_custom_gcode]`. Advanced / rarely used. **Leave unset.**

---

## Summary of recommended Machine G-code tab changes

### Per-setting decisions

| Setting | Current | Decision | Why |
|---|---|---|---|
| **Start G-code** | `PRINT_START EXTRUDER_TEMP=... BED_TEMP=...` (override) | **no change** | Override is *better* than default — parallelises heating with homing/mesh, saves 1-3 minutes per print |
| **End G-code** | `PRINT_END` (inherited from `fdm_klipper_common`) | **no change** | Correct — delegates all print-end logic to the Klipper macro, single source of truth |
| **Layer change G-code** | `;AFTER_LAYER_CHANGE\n;[layer_z]` (inherited) | **no change** | Comment-only marker. Correct default for "no custom per-layer action" |
| **Before layer change G-code** | `;BEFORE_LAYER_CHANGE\n;[layer_z]\n` (override drops `G92 E0`) | **no change** | Defensible — our PRINT_START sets `M83` (relative E), so `G92 E0` is a no-op. Harmless |
| **Pause G-code** | `PAUSE` (inherited) | **no change** | Maps to Klipper's built-in PAUSE command. Correct |
| **Change filament G-code** | empty (inherited) | **no change** | N/A for single-extruder — we don't do multi-material swaps |
| **Time lapse G-code** | empty | **no change (for now)** | Optional: set to `TIMELAPSE_TAKE_FRAME` if you want moonraker-timelapse video. Leave empty unless you actively want timelapses |
| **Template custom G-code** | not set | **no change** | Advanced / rarely used placeholder |

### Strongly recommended

**None.** This tab is in good shape — all overrides are correct and beneficial, and the defaults that are inherited are the right defaults.

### Optional refinements

| Setting | Currently | Could change to | Reason |
|---|---|---|---|
| `time_lapse_gcode` | (empty) | `TIMELAPSE_TAKE_FRAME` | Only if you want to use moonraker-timelapse for per-print video. Leave empty if not |
| `macros.cfg` PRINT_START last line | `PRIME_LINE` (hardcoded prime line at X10 Y10→Y200) | `LINE_PURGE` (KAMP adaptive prime line adjacent to first object) | KAMP's LINE_PURGE is smarter and already installed. **This is a Klipper-side change, not slicer** |
| `macros.cfg` PRINT_END fan off | Commented out `M107 S0` | Uncomment | Explicit is better than relying on slicer/firmware defaults. Minor. **Klipper-side change** |

### Confirmed correct (no change needed)

- `machine_start_gcode` — our override parallelises heating with homing/mesh. **Beneficial, leave as-is**
- `machine_end_gcode` — `PRINT_END` (inherited from Klipper base). **Correct**
- `layer_change_gcode` — comments only (inherited). **Correct**
- `before_layer_change_gcode` — our override drops G92 E0. **Harmless in M83 mode, leave as-is**
- `machine_pause_gcode` — `PAUSE` (inherited from Klipper base). **Correct**
- `change_filament_gcode` — empty (inherited). **Correct for single-extruder**

---

## How the slicer and Klipper macros interact (conceptual reference)

This is a useful mental model to have when reading/editing anything on this tab:

```
╔══════════════════╗          ╔══════════════════╗
║   OrcaSlicer     ║          ║     Klipper      ║
║  (slicer side)   ║  gcode   ║ (firmware side)  ║
╠══════════════════╣ ──────►  ╠══════════════════╣
║ machine_start_   ║          ║ gcode_macro      ║
║ gcode            ║          ║ PRINT_START      ║
║                  ║          ║   (macros.cfg)   ║
║ "PRINT_START     ║          ║   - FANS_ON      ║
║  EXTRUDER_TEMP=  ║          ║   - M140/M104    ║
║  [value]         ║          ║   - G28          ║
║  BED_TEMP=       ║  ─────►  ║   - Z_TILT       ║
║  [value]"        ║          ║   - BED_MESH     ║
║                  ║          ║   - M190/M109    ║
║                  ║          ║   - PRIME_LINE   ║
╚══════════════════╝          ╚══════════════════╝
```

The **slicer side** is a thin bridge: "call the macro with these params". The **Klipper side** is where all the actual logic lives — homing, mesh, tilt, prime line, end moves. This means:

1. **Changes to print logic** (new prime line strategy, different end park position, chamber heating, pre-print purge) → edit `macros.cfg` on the printer, **not** the slicer
2. **Changes to temperatures** → automatic via slicer's `[nozzle_temperature_initial_layer]` / `[bed_temperature_initial_layer_single]` placeholders
3. **Changes to layer marker behaviour** (timelapse triggers, per-layer chamber checks, etc.) → edit `layer_change_gcode` / `before_layer_change_gcode` in the slicer

**Keeping logic in Klipper is the right call** because:

- The same logic applies regardless of slicer (OrcaSlicer, PrusaSlicer, SuperSlicer)
- The Klipper config is the source of truth for how this specific printer behaves
- Modifying a macro doesn't require re-slicing

---

## Cross-reference

- **[`basic_information.md`](basic_information.md)** — the `PRINT_END` park position `X260 Y240` needs to match the `printable_area` setting; that's covered in the basic info tab
- **[`motion_ability.md`](motion_ability.md)** — the `machine_start_gcode` doesn't touch motion limits directly but the bed_mesh / Z tilt moves use the Z-axis limits that tab covers
- **[`extruder.md`](extruder.md)** — the 8mm end-retract in `PRINT_END` is independent of slicer retraction settings (the slicer's retract fires on retractions during the print; the end retract is a one-shot firmware-side move)
- **[`../process/others_PLA.md`](../process/others_PLA.md)** — contains the recommendation to add the klipper_estimator post-processing line to slicer's post-processing field, which modifies the gcode **after** the slicer emits it but **before** Klipper executes it

---

## References

[^klipper-macros]: Klipper `[gcode_macro]` reference — how Klipper parses custom macros, parameter handling via `params.NAME|default(X)|float`, and the `{% set ... %}` templating syntax used in `PRINT_START`. <https://www.klipper3d.org/Command_Templates.html>

[^kamp-line-purge]: KAMP's adaptive line purge replaces the hardcoded PRIME_LINE with a prime line drawn adjacent to the first printed object. Benefits: less wasted filament, no collision risk with prints that occupy the hardcoded prime area. Already installed on our printer at `~/printer_data/config/KAMP/Line_Purge.cfg` but not wired into PRINT_START yet. <https://github.com/kyleisah/Klipper-Adaptive-Meshing-Purging>

[^moonraker-timelapse]: Moonraker-timelapse is the standard Klipper timelapse plugin. It defines `TIMELAPSE_TAKE_FRAME` which captures a camera frame; Moonraker stitches frames into an MP4 at print end. Pre-installed on MainsailOS. Config file: `~/printer_data/config/timelapse.cfg`. Trigger macro must be added to slicer's Time Lapse G-code for it to fire. <https://github.com/mainsail-crew/moonraker-timelapse>

[^relative-e]: OrcaSlicer defaults to **relative E distance mode** (`M83`), where each `G1 E0.4` means "extrude 0.4mm more" rather than "move E axis to absolute position 0.4". In relative mode, the `G92 E0` reset command is practically redundant because the slicer never references the growing internal counter. Klipper's double-precision position tracking also eliminates the float32 precision concern that motivated G92 E0 in Marlin. <https://www.klipper3d.org/G-Codes.html#m82-m83-extruder-absolute-or-relative-mode>

[^orca-macros-tab]: OrcaSlicer Machine G-code tab reference — defines which gcode placeholders are available (`[nozzle_temperature_initial_layer]`, `[bed_temperature_initial_layer_single]`, `[layer_z]`, `[layer_num]`, etc.) and when each gcode section fires. <https://github.com/SoftFever/OrcaSlicer/wiki/Custom-G-code>
