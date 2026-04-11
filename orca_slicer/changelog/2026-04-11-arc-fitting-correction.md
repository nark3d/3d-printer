# Arc fitting correction — 2026-04-11

> Correction to change #19 from the full settings cleanup pass. The original recommendation was wrong; arc fitting should be OFF on Klipper, not ON.

## What changed

**Single change**: `enable_arc_fitting: "1"` → `"0"` in `0.20mm Standard @MyKlipper - PLA.json`.

## Backup

The pre-correction state was copied to `orca_slicer/backups/2026-04-11-arc-fix/0.20mm Standard @MyKlipper - PLA.json` before editing.

## Why the original recommendation was wrong

The full-settings-cleanup changelog change #19 said:

> **`enable_arc_fitting`: false → true**
> VzBot bundled ships this enabled. Klipper supports G2/G3 arcs (we have `[gcode_arcs]` in `printer.cfg`) — the "Klipper doesn't support arcs" claim from one source is wrong.

Both halves of that reasoning were flawed:

### 1. ERS mutual exclusion (blocker — setting can't even be enabled)

**OrcaSlicer's UI forcibly disables `enable_arc_fitting` whenever `max_volumetric_extrusion_rate_slope > 0`.**

Our profile has ERS at 300 mm³/s² (Adam L's value from the scarf seam research, field-validated on our hardware). This is a high-quality setting we deliberately want to keep.

**Result**: even with `enable_arc_fitting: "1"` in the JSON, the UI shows the Arc fitting checkbox greyed out. Toggling it produces an orange "differs from preset" circle but the checkbox can't be enabled. Every time OrcaSlicer saves the profile, it removes the `enable_arc_fitting: "1"` key or resets it to inherit-default false.

This caused a recurring "unsaved changes" dialog on every New Project action, displaying `Arc fitting: Old value true, New value false`. Clicking Save on that dialog (incorrect earlier advice) wrote the stale UI-cached state back to disk, **stripping 16 other keys from our profile** as a side effect. This required a full profile restoration from backup.

**Source**: [OrcaSlicer Issue #4216](https://github.com/SoftFever/OrcaSlicer/issues/4216) — project collaborator quote: "ERS does not play ball with Arc fitting. so it is disabled."

### 2. Arc fitting is discouraged for Klipper even WITHOUT ERS

Even if we disabled ERS, enabling arc fitting on Klipper creates a **double lossy conversion**:

1. Slicer generates perimeter as linear segments internally (lossless)
2. `enable_arc_fitting` converts those to G2/G3 arcs (**lossy step 1** — geometric approximation error, typically 0.025 mm tolerance)
3. Gcode written to file with G2/G3
4. Klipper parses G2/G3 via `[gcode_arcs]` module
5. Klipper converts them back to line segments via `resolution: 1 mm` default (**lossy step 2** — segment length approximation)
6. Klipper motion planner executes line segments

**Source**: [OrcaSlicer Issue #4216](https://github.com/SoftFever/OrcaSlicer/issues/4216) — "Arc fitting is discouraged for Klipper-based machines, as Klipper converts arc commands back to line segments, resulting in quality loss from two sets of lossy conversions." PR #5352 updated OrcaSlicer docs to make this explicit.

**Community confirmation**: [Creality forum — Arc Fitting in process profiles for Klipper based machines](https://forum.creality.com/t/arc-fitting-in-process-profiles-for-klipper-based-machines/26232) — same conclusion for all Klipper-based printers.

## The ZeroG documentation confusion (cleared up)

The user noted that ZeroG's documentation recommends "using arcs" with Klipper — which seemed to contradict the "don't enable arc fitting" advice. They don't contradict; they're about different settings in different systems:

| Recommendation | What it means | Where it lives | Status |
|---|---|---|---|
| ZeroG: "enable arcs in Klipper" | Enable the `[gcode_arcs]` module in `printer.cfg` so Klipper can **parse** G2/G3 commands | `~/printer_data/config/printer.cfg` on the Pi | ✅ already enabled (verified via Moonraker: `[gcode_arcs]` section exists) |
| Community: "disable arc fitting in slicer" | Don't have the slicer **produce** G2/G3 commands — leave the perimeter as G1 line segments | OrcaSlicer process profile | ✅ now `enable_arc_fitting: "0"` |

These are not contradictory — they work together. `[gcode_arcs]` in Klipper lets the firmware *handle* any arcs that happen to appear (e.g. from CAM software or external gcode). But for normal printing, you want the slicer to *produce* line segments directly so Klipper doesn't have to do its own line-to-arc-to-line conversion round trip.

We have the correct combination now:
- Klipper printer.cfg: `[gcode_arcs]` enabled (ZeroG recommendation ✓)
- OrcaSlicer profile: `enable_arc_fitting: "0"` (community recommendation for Klipper ✓)

## What was lost by making this correction

**Nothing.** Arc fitting on Klipper produces lossy-conversion quality reduction, not improvement. By setting it to `0`:

- ✅ ERS stays at 300 (high-quality field-validated setting)
- ✅ Klipper still handles arcs if any external gcode has them (`[gcode_arcs]` still on)
- ✅ No more dialog on every New Project
- ✅ No more orange circle in the UI
- ✅ No more risk of the profile being accidentally stripped on save
- ✅ Slightly better curve quality (single lossless line-segment path, no double approximation)

The only "loss" is the gcode file size — with arc fitting, a typical print might have been 10-30% smaller. Without it, the file is larger. **This doesn't affect print quality or time in any measurable way on Klipper.**

## Documentation updates

Updated files:
- `orca_slicer/process/quality_PLA.md` — corrected the Arc fitting row in the Precision table with full explanation, ERS note, Klipper note, and ZeroG clarification
- `orca_slicer/process/quality_PLA.md` — corrected change #19 in the summary table
- `orca_slicer/process/quality_PLA.md` — added footnote `[^arc-fitting-klipper]` with research trail to Issue #4216 and Creality forum

The original changelog entry in `2026-04-11-full-settings-cleanup.md` is **not edited** — it stays as a historical record of what was attempted. This correction file is the authoritative current state.

## Key takeaway for future settings work

**When a bundled profile (like VzBot) has a setting that seems obviously right, check whether it applies to our specific firmware.** VzBot bundled is tuned for the VzBot 330 AWD, which may or may not be running Klipper — and even if it is, quality-related settings don't automatically transfer. The VzBot profile had `enable_arc_fitting: true`, which may be correct for their hardware combination, but is wrong for Klipper + `[gcode_arcs]`-style setups because of the double-conversion issue.

**Lesson**: firmware-dependent settings (arc fitting, gcode flavor, extrusion advance model) need explicit cross-checking against the target firmware's documentation, not just "copy from the well-maintained bundled profile".

## Grand total for today after this correction

**55 total changes** across the three profiles:
- Printer profile: 16 changes
- Process profile: **31 changes** (16 cleanup + 5 discrepancy + 3 ironing + 9 seam + 1 arc-fitting correction — change #19 reversed)
- Filament profile: 5 changes

## Rollback for this correction only

If for some reason we want to restore the pre-correction state (with `enable_arc_fitting: "1"`, which will be immediately stripped by OrcaSlicer again):

```bash
cp orca_slicer/backups/2026-04-11-arc-fix/"0.20mm Standard @MyKlipper - PLA.json" \
   "/Users/user/Library/Application Support/OrcaSlicer/user/default/process/"
```

This is not recommended — the correction is the right state.
