# OrcaSlicer printer settings — ZeroG Mercury One profile

> One markdown file per OrcaSlicer printer settings tab. Each file lists every setting on that tab with: the OrcaSlicer default, the researched community value (with footnote citations), our current value, and a comments field documenting the reasoning behind any deviation. Same format as the [`process/`](../process/README.md) tab files.
>
> **Critical difference from process settings**: many printer settings reflect the *actual hardware geometry* (printable area, extruder clearance, nozzle diameter) and therefore have a *correct* answer, not a "preference" answer. When the OrcaSlicer value disagrees with reality (which we'll see in several places), the slicer plans gcode based on the wrong reality and produces sub-optimal output. **Fixing those mismatches is the highest-value work in this folder.**

## Tab files

| Tab | File | Status |
|---|---|---|
| **Basic information** | [`basic_information.md`](basic_information.md) | ✅ Researched, 10 references |
| **Machine G-code** | [`machine_gcode.md`](machine_gcode.md) | ✅ Researched, 4 references |
| **Extruder** | [`extruder.md`](extruder.md) | ✅ Researched — critical wipe/retract finding flagged |
| **Motion ability** | [`motion_ability.md`](motion_ability.md) | ✅ Researched — highest-impact tab, 4× accel mismatch flagged |
| **Multimaterial** | (skipped) | ⛔ Adam: "we don't really need multimaterial" |
| **Notes** | [`notes.md`](notes.md) | ✅ Researched — optional, no functional effect |

## Top findings across all tabs

If you only fix three things from the printer settings research, these are the ones:

| # | Tab | Finding | Fix |
|---|---|---|---|
| 1 | Motion ability | All `machine_max_acceleration_*` values are **4× higher** than what Klipper `printer.cfg` actually enforces (40 000 vs 10 000). Slicer plans gcode for headroom that doesn't exist | Set `machine_max_acceleration_x/y/extruding/retracting/travel` to `10 000, 5 000` to match Klipper |
| 2 | Extruder | Both `retract_when_changing_layer` and `wipe` are **off** — exactly the combination Adam L's "Better Seams" guide warns against. This causes visible seam gorges | Turn **both ON** |
| 3 | Basic information | `printable_area` is 260×240 but Klipper `printer.cfg` has Y=245, leaving **5 mm of build area unused** | Set `printable_area` to match Klipper: 260×245 |

See each tab document for full reasoning and cross-references.

## Hardware context (shared across all tabs)

- **Frame / kinematics**: ZeroG Mercury One.1 + Nebula Pro frame (CoreXY, **260 × 245 × 258 mm** build volume — confirmed against `printer.cfg`)
- **Toolhead**: VzBot CNC toolhead (aluminium variant)
- **Hotend**: Phaetus Rapido 2 HF (high-flow)
- **Extruder**: Orbiter 2.5 (7.5 : 1 geared, ~0.06 mm measured backlash)
- **Nozzle**: 0.4 mm hardened steel
- **Probe**: Beacon RevH (sub-micron Z probe σ measured)
- **Klipper motion limits** (from `printer.cfg`): `max_velocity` 600 mm/s, `max_accel` 10 000 mm/s², `square_corner_velocity` 5 mm/s
- **Input shaper**: MZV X=61.0 Hz / Y=42.0 Hz (LIS2DW-derived)

## Inheritance chain

Our profile is `ZeroG Mercury One.json` and resolves through:

```
ZeroG Mercury One                           (user, 31 keys)
  → MyKlipper 0.4 nozzle                    (system Custom, 11 keys, sets nozzle 0.4 + printable area 250×250)
    → fdm_klipper_common                    (Klipper base, 58 keys)
      → fdm_machine_common                  (machine base, ~30 more keys)
```

Each tab file's "OrcaSlicer default" column is the value Orca arrives at *before* any of our overrides, walking that chain.

## Cross-reference: Klipper-side configuration

Several printer settings need to **match** the Klipper `printer.cfg` to avoid the slicer planning gcode based on wrong assumptions. The canonical Klipper config is at `~/printer_data/config/printer.cfg` on the printer (referenced in [`HARDWARE.md`](../../HARDWARE.md)). If you change a Klipper-side value (e.g. `max_velocity`), update the matching OrcaSlicer printer setting to keep them in sync.

## Column meanings

Same as the process tab files:

- **Setting** — UI label as shown in OrcaSlicer
- **OrcaSlicer default** — value Orca arrives at without our overrides
- **Researched value** — community references / vendor docs / Klipper docs, with footnote citations
- **Our value** — what our `ZeroG Mercury One.json` profile currently has set
- **Comments** — reasoning for our choice
