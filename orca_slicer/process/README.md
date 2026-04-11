# OrcaSlicer process settings — PLA quality profile

> One markdown file per OrcaSlicer process tab. Each file lists every setting on that tab with: the OrcaSlicer default, the researched community value (with footnote citations), our current value, and a comments field documenting the reasoning behind any deviation. Researched values come from a mix of: the OrcaSlicer wiki, OrcaSlicer-bundled vendor profiles (notably the VzBot one), GitHub issues and discussions in the OrcaSlicer/SoftFever repos, the Bambu Lab wiki, the Andrew Ellis Print Tuning Guide, and community-shared profiles on Printables/MakerWorld.

## Tab files

| Tab | File | Status |
|---|---|---|
| **Quality** | [`quality_PLA.md`](quality_PLA.md) | ✅ Researched, 24 references |
| **Strength** | [`strength_PLA.md`](strength_PLA.md) | ✅ Researched, 8 references |
| **Speed** | [`speed_PLA.md`](speed_PLA.md) | ✅ Researched, 7 references |
| **Others** | [`others_PLA.md`](others_PLA.md) | ✅ Researched, 4 references |
| **Support** | (skipped) | ⛔ Adam: "we don't really need support" |
| **Multimaterial** | (skipped) | ⛔ Adam: "we don't really need multimaterial" |

## Topic-specific deep dives

For complex topics that span multiple settings or warrant standalone treatment:

| Topic | File | Description |
|---|---|---|
| **Seams & scarf seams (theoretical reference)** | [`seams_deep_dive.md`](seams_deep_dive.md) | The complete reference based on Adam L's "Better Seams" Printables guide, the OrcaSlicer wiki, the original PR #3839, and community sources. Covers every scarf-related setting in depth, edge cases, filament considerations, tuning workflow. **Read this when you want to understand seams in general.** |
| **Seams (empirical field notes)** | [`seam_diagnosis_field_notes.md`](seam_diagnosis_field_notes.md) | Adam's hard-won diagnostic findings from real prints over the past weeks of testing. Covers specific failure modes (F120 dead-stop, seam gorge, scarf knotty blob), the 2024 known-good profile, per-filament PA values, and key principles learned. **Read this when you're diagnosing a real seam problem on a real print.** |
| **Ironing (top-surface smoothing)** | [`ironing_deep_dive.md`](ironing_deep_dive.md) | Complete reference for top-surface ironing: physics, every setting, interaction with other quality settings, calibration workflow, gotchas, per-material tables (PLA calibrated, ASA/PETG/ABS/TPU placeholders). 22 references. **Read this when you want to understand or tune ironing.** |

## Hardware context (shared across all tabs)

- **Frame / kinematics**: ZeroG Mercury One.1 + Nebula Pro frame (CoreXY, 260 × 245 × 258 build volume) — *not* a VzBot 330, despite using a VzBot toolhead
- **Toolhead**: VzBot CNC toolhead (aluminium variant, heavier than the printed Vz-Printhead the OrcaSlicer-bundled VzBot profile is calibrated for)
- **Hotend**: Phaetus Rapido 2 HF (high-flow, claimed 45 mm³/s; practical PLA ~22-28 mm³/s)
- **Extruder**: Orbiter 2.5 (7.5 : 1 geared, ~0.06 mm measured backlash)
- **Probe**: Beacon RevH (sub-micron Z probe σ measured)
- **Motion limits**: `max_velocity` 600 mm/s, `max_accel` 10 000 mm/s², input shaper MZV X=61.0 Hz / Y=42.0 Hz (LIS2DW-derived)
- **Material**: PLA, any brand
- **Goal**: Quality-first

## Why "VzBot toolhead, not VzBot printer" matters

The OrcaSlicer-bundled `0.20mm Standard @Vzbot` process profile is calibrated for the *whole* VzBot 330 AWD with the *lightweight* Vz-Printhead (~22-27 g aluminium head). Our setup uses the heavier CNC toolhead variant on a Mercury One.1 frame. Direct consequences:

| Category | Bundled VzBot profile is… |
|---|---|
| Hotend flow / volumetric speed | Directly applicable (same hotend) |
| Extruder retraction / pressure advance | Directly applicable (same extruder) |
| Quality (line widths, walls, seams, ironing, bridging flow) | Mostly applicable (toolhead-mass-independent) |
| Strength (wall counts, infill patterns, shell layers) | Mostly applicable (geometry-driven, not toolhead-mass) |
| Speeds | An *upper bound* — heavier toolhead → slower viable speeds |
| Accelerations | An *upper bound* — lower-frequency input shaper means less accel headroom |
| Frame / kinematics options | Not applicable — different printer |

## Inheritance reminder

Our PLA profile resolves through:

```
0.20mm Standard @MyKlipper - PLA   (our overrides, ~46 keys)
  → 0.20mm Standard @MyKlipper      (only sets layer_height)
    → fdm_process_klipper_common    (Klipper-specific overrides, 16 keys)
      → fdm_process_common          (the base, 106 keys)
```

The "OrcaSlicer default" column in each tab file is the value Orca arrives at *before* any of our overrides, walking that chain from `fdm_process_common`.

## Column meanings

- **Setting**: UI label as shown in OrcaSlicer
- **OrcaSlicer default**: Value Orca arrives at without overrides. Where I couldn't confirm with confidence I've marked it explicitly (`?` or `(probably X)`) rather than fabricate
- **Researched value**: What community references / vendor docs / curated profiles recommend, with footnote citations to the source
- **Our value**: What our `0.20mm Standard @MyKlipper - PLA` profile currently has set. `(default)` means we don't override and inherit the OrcaSlicer default
- **Comments**: Reasoning for our choice — especially when we differ from either the default or the researched value
