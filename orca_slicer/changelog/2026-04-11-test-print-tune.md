# Test print analysis — tuning pass — 2026-04-11

> Four-setting tuning pass based on empirical observations from the 40 mm / 30 mm annulus test print. Applied in a single batch per the user's "apply all, then analyse" request. Subsequent test print will confirm or refute each change.

## Test print observations that drove these changes

User printed a 40 mm OD / 30 mm ID / 10 mm tall annulus with `seam_slope_type: external` (temporary one-off override for this test — scarf on outer cylinder, traditional seam on hole) to get a side-by-side comparison of scarf vs traditional seams + ironed top.

**Print quality observations (user report + photos)**:
1. **Scarf seam on outer wall**: excellent overall, but visible discrete step bands (user counted "20 of them")
2. **Traditional seam on hole**: visible dip + step (feelable with fingernail), user asks how to reduce it
3. **Ironing on top annulus**: two drag marks visible on the ironed surface
4. **Print time**: Spiral Z hop adds noticeable overhead

## Reference photos with issue annotations

Photos taken by the user of the test prints. The raw WhatsApp images were dark red/purple PLA (Sunlu Matte), which washes out detail — so the "grayscale" versions have had colour stripped and contrast boosted to expose the artefacts.

**Image paths use absolute `file://` URLs** for maximum viewer compatibility. Files live in `/Users/user/Repos/3d-printer/orca_slicer/changelog/images/2026-04-11-test-print-tune/`.

### Test print #1 — initial baseline (seam_slope_type: external)

Annulus printed with scarf on outer wall, traditional seam on hole interior — to see both seam types side-by-side on one print.

#### Photo 1 — Scarf seam 20 bands on outer wall (Issue #1)

<img src="file:///Users/user/Repos/3d-printer/orca_slicer/changelog/images/2026-04-11-test-print-tune/01-scarf-seam-outer-wall-grayscale.jpeg" alt="Scarf seam bands grayscale" width="700">

**What to look for**: on the **outer wall** (left side of ring), subtle horizontal banding in the scarf slope region — ~20 discrete steps stacked into the wall. User counted "all 20 of them", matching `seam_slope_steps: 20`.

**Hypothesis (change #55)**: these are NOT the Z-step stair marks (9 microns/step, invisible). They're **feedrate transition bands from ERS** smoothing the flow change. Reducing `max_volumetric_extrusion_rate_slope` from 300 → 150 should flatten them into a smoother gradient.

Raw colour image: `04-original-close-up-colour.jpeg` in the same folder.

#### Photo 2 — Traditional seam on hole interior + ironing drag marks on top (Issues #2 & #3)

<img src="file:///Users/user/Repos/3d-printer/orca_slicer/changelog/images/2026-04-11-test-print-tune/02-traditional-seam-and-ironing-top-down-grayscale.jpeg" alt="Traditional seam + ironing grayscale top-down" width="700">

**Top-down view looking into the hole. Shows TWO issues at once.**

**Issue #2 — Traditional seam (look at the inner wall of the hole)**: vertical dark line running top-to-bottom on the **left side of the hole interior**. User feels a dip + one-layer-height step with a fingernail. This is the traditional seam on the hole perimeter (caused by `seam_slope_type: external` for this test — scarf only on outer wall).

**Fix attempted (change #58)**: bump `filament_retract_restart_extra` from 0.03 → 0.10 on the Sunlu Matte filament profile. Theory: extra material after de-retract fills the PA-caused dip at loop start.

**Issue #3 — Ironing drag marks (look at the flat top surface of the annulus ring)**: faint horizontal striations. Some are the rectilinear ironing pattern (expected), but two are deeper lines — drag marks where the nozzle wipe-retract sequence scraped material during the ironing pass.

**Fix (change #56)**: halve printer-side `wipe_distance` from 2 → 1 mm. Gcode analysis (lines 97683-97716) confirmed: each ironing retract emits a 2 mm wipe at F2400, **at layer Z**, before Spiral Lift begins. Literally dragging the nozzle across the ironed surface. Halving the distance halves the drag.

Raw colour image: `03-original-top-down-colour.jpeg` in the same folder.

---

### Test print #2 — after applying all 4 changes

Same annulus re-printed with the 4 settings changes (ERS 150, wipe_distance 1, z_hop 0.2, filament_retract_restart_extra 0.10). Tested with `seam_slope_type: external` again to maintain the side-by-side comparison.

**Result: partial success, one regression.**

| Issue | Outcome | Notes |
|---|---|---|
| Scarf 20 bands | Not fully resolved (visible change but not eliminated) | Need to evaluate further |
| Traditional seam dip on hole | ✅ **Good** — user confirmed "the normal seam is nice, it feels good on my thumbnail" | The 0.10 restart_extra worked for traditional seams |
| Ironing drag marks | Partial improvement — still visible, user reports non-contiguous ironing pattern | `wipe_distance: 1` helped but didn't eliminate; the root cause is the rectilinear pattern's inter-segment travels |
| Print time | Improved | From z_hop reduction |
| **NEW REGRESSION: scarf seam bulge** | 🔴 **Visible vertical bulge at scarf entry** running the full height of the outer wall | Caused by `filament_retract_restart_extra: 0.10` |

#### Photo 3 — The scarf seam bulge (NEW regression from change #58)

<img src="file:///Users/user/Repos/3d-printer/orca_slicer/changelog/images/2026-04-11-test-print-tune/test2-03-scarf-bulge-grayscale.jpeg" alt="Scarf seam bulge after restart_extra change" width="700">

**What to look for**: a visible **vertical bulge/ridge** on the **left side of the outer wall**, running the full 10 mm height of the annulus. This is a raised artefact — material piled up, not missing.

**Root cause**: `filament_retract_restart_extra: 0.10` pushes 0.10 mm of filament into the nozzle at every de-retract. At a scarf entry, the nozzle sits **0.02 mm above the previous layer** (10% of layer height, per `seam_slope_start_height: 10%`). Dumping ~0.24 mm³ of PLA into a 0.02 mm clearance gap creates a blob. Stacked across 50 layers → visible vertical bulge.

**This is a fundamental conflict**: `filament_retract_restart_extra` fires at every de-retract regardless of whether the next feature is scarf or traditional. OrcaSlicer has no per-feature override. The same setting that **helped the traditional seam** (fills the dip) **broke the scarf** (creates a bulge at the ramp start).

#### Photo 4 — Bulge from a different angle (confirms mechanism)

<img src="file:///Users/user/Repos/3d-printer/orca_slicer/changelog/images/2026-04-11-test-print-tune/test2-04-scarf-bulge-grayscale-alt.jpeg" alt="Scarf bulge from alt angle" width="700">

Same ring, different angle. You can see the bulge pattern more clearly from this view — it's a thick vertical stripe running the full height of the wall, right where each layer's scarf entry point lands.

Raw colour images: `test2-01-original-scarf-bulge-angle-1.jpeg`, `test2-02-original-scarf-bulge-angle-2.jpeg` in the same folder.

---

## Changes in test-print-tune round 2 — 2026-04-11

Applied after test print #2 showed the bulge regression and the user's feedback that traditional seams should remain viable (can't just switch everything to scarf).

### 59. `filament_retract_restart_extra`: `0.10` → `0.05` (Sunlu Matte filament)

**Why**: full revert to 0.03 loses the traditional seam improvement. Full 0.10 causes the scarf bulge. **0.05 is a middle-ground test value** — halves the extra material at scarf entries (hopefully making the bulge invisible or minimal) while still providing some fill to the traditional seam dip.

**Uncertainty**: this value might not be optimal for either. May need further iteration (0.04, 0.06, 0.07) based on test results.

**Alternative considered**: per-feature `retract_restart_extra` would be ideal but OrcaSlicer doesn't support it — no path to having different values for scarf vs traditional on the same print.

### 60. `outer_wall_line_width`: `105%` → `150%` (process profile)

**Why**: Adam L's specifically tested value for best scarf seam quality. Wider lines mean each mm of perimeter has more material, so flow variation at seams is a smaller percentage of total material → less visible. Benefits BOTH scarf and traditional seams via the same mechanism.

**Source**: [Better Seams Printables guide](https://www.printables.com/model/783313-better-seams-an-orca-slicer-guide-to-using-scarf-s) by Adam L — 140+ experiments on Bambu X1C.

**Cost**: ~0.1 mm dimensional accuracy hit on holes. User explicitly accepted this: "Happy to try the 150% wall, the accuracy deficit on holes only is acceptable."

### 61. `ironing_pattern`: `rectilinear` → `concentric` (process profile)

**Why**: the user's observation that ironing is "not a contiguous pattern, it's lifting and moving" is correct. Rectilinear ironing on an annulus creates many short parallel-line segments with travels between them — each travel > 3 mm triggers retract + wipe + Z hop.

**Research finding**: [OrcaSlicer discussion on pattern continuity](https://github.com/SoftFever/OrcaSlicer/discussions/66) + [Issue #3254 on only_retract_when_crossing_perimeters](https://github.com/SoftFever/OrcaSlicer/issues/3254) confirm:
1. OrcaSlicer's "Concentric" ironing is nested rings, NOT a single continuous spiral (that's a still-open feature request)
2. However, for an **annulus** geometry, concentric creates ~50 rings at `ironing_spacing: 0.1 mm` apart — each ring is continuous AND the inter-ring travel (0.1 mm) is below `retraction_minimum_travel: 3 mm` so **no retract fires, no wipe, no Z hop, no drag marks**
3. OrcaSlicer removed the `only_retract_when_crossing_perimeters` option so we can't use that lever

**Expected result on annulus**: the rings spiral inward from outer edge toward inner edge with only 0.1 mm gaps between them. Each ring is cleanly ironed, and the inter-ring transitions are below the retraction threshold. **No drag marks**.

**Trade-off on other geometries**:
- **Solid circular top**: concentric has a known "center dot" artefact where the spiral terminates. For our typical prints (functional parts, mechanical features), this is usually less visible than rectilinear drag marks.
- **Square tops**: concentric traces rectangular rings inward. Works but less geometrically efficient than rectilinear. On very large flat square tops, the corners of the concentric rings can look slightly uneven.

The user's actual print targets (mostly functional parts with varied geometry) benefit more from no-drag concentric than from rectilinear efficiency.

**`ironing_angle_fixed: 1` + `ironing_angle: 90`** remain set but are effectively **ignored** by concentric pattern (rings don't have a single angle). Harmless.

---

## Settings state after round 2 (applied)

| Setting | File | Value |
|---|---|---|
| `max_volumetric_extrusion_rate_slope` | process | 150 *(from round 1)* |
| `wipe_distance` | printer | 1 *(from round 1)* |
| `z_hop` | printer | 0.2 *(from round 1)* |
| `filament_retract_restart_extra` | filament (Sunlu Matte) | **0.05** *(round 2 — was 0.10)* |
| `outer_wall_line_width` | process | **150%** *(round 2 — was 105%)* |
| `ironing_pattern` | process | **concentric** *(round 2 — was rectilinear)* |

## Settings NOT changed this round

- `seam_slope_type` stays at `external` in the UI override for continued side-by-side testing. **User explicitly wants both seam types on one print** to keep improving traditional seams; will not switch to `all` as previously suggested. This is the right approach — it's how real prints have mixed features.
- ERS, wipe_distance, z_hop stay from round 1 — no evidence they need changing yet.

## Expected outcomes for test print #3

| Issue | What round 2 should change |
|---|---|
| **Scarf bulge (regression from round 1)** | **Primary target** — should be significantly reduced with restart_extra at 0.05. May still be slightly visible; fine-tune further if needed |
| **Traditional seam dip** | Should remain improved but not as much as 0.10. Trade-off accepted |
| **Scarf 20 bands** | Should be further improved by the wider 150% outer wall line width |
| **Ironing drag marks** | Should be largely eliminated by concentric pattern (no retracts within ironing) |
| **Print time** | Similar to round 1 |

---

### Processing note

Both grayscale images were generated from the raw WhatsApp JPEGs via Python + PIL:
- Crop to relevant region of the ring
- Convert to greyscale (removes distracting Sunlu Matte colour)
- `ImageOps.autocontrast` with 2% cutoff (histogram stretch)
- 1.3× contrast enhancement
- 1.5× sharpness enhancement

The raw colour photos are preserved in the same folder for reference (`03-original-top-down-colour.jpeg`, `04-original-close-up-colour.jpeg`).

## Backups

All three profiles backed up to `orca_slicer/backups/2026-04-11-test-print-tune/` before editing.

## Changes applied (4 total)

### 55. `max_volumetric_extrusion_rate_slope`: `300` → `150` (process profile)

**Why**: the visible "20 step bands" on the scarf seam are likely **feedrate-change bands from ERS**, not the 20 Z stair-steps (at 9 microns per Z step, well below visible threshold). Gcode analysis of the previous test print showed 11 distinct feedrate transitions during the scarf slope (F3000 → F4500 over 20 segments), caused by ERS progressively ramping velocity as flow changes.

Lowering ERS to 150 mm³/s² (Voron community standard) means **smoother, fewer, wider-spaced feedrate transitions** — the scarf slope should read as a single gradient rather than 20 discrete bands.

Tradeoff: ERS is slightly less aggressive at smoothing rapid flow changes elsewhere (e.g. overhangs transitioning to infill). This is the Voron community default, so it should be fine on our hardware.

**Source**: Gcode inspection of test print (lines 2555-2598 show the scarf slope feedrate ramping), community consensus per [seams_deep_dive.md § 12.8](../process/seams_deep_dive.md), Adam L originally used 300 on Bambu X1C (not directly transferrable).

### 56. `wipe_distance`: `2` → `1` (printer profile)

**Why**: ironing drag marks. Gcode analysis (lines 97683-97716) confirmed every ironing retract emits a 2 mm wipe move at F2400 (40 mm/s) **at layer Z**, BEFORE the Spiral Lift Z hop begins. That's the nozzle physically dragging across the ironed top surface for 2 mm during each retract.

Halving the wipe distance to 1 mm halves the drag distance per retract. Each ironing line has ~2 wipe events, so this affects hundreds of moves on a typical ironed print.

Adam L's "do not" combination is `retract_when_changing_layer: 0 + wipe: 0` (both off). We keep `wipe: 1` and just reduce its distance — Adam L's rule is satisfied.

**Source**: Direct gcode observation, our own [`seams_deep_dive.md`](../process/seams_deep_dive.md), [`seam_diagnosis_field_notes.md`](../process/seam_diagnosis_field_notes.md).

### 57. `z_hop`: (inherited 0.4) → explicit `0.2` (printer profile)

**Why**: print time. Our `max_z_velocity: 5 mm/s` means every Z hop takes `2 × (z_hop / 5) × 1000 = 160 ms` round-trip for 0.4 mm hop. The previous test print had 974 wipe/hop sequences — roughly **156 seconds of Z hop time** on a ~15 minute print (~17%).

Halving `z_hop` to 0.2 mm halves the per-hop time (80 ms round-trip), saving ~78 seconds on the same print while still clearing the nozzle above most surface artefacts.

We keep `z_hop_types: Spiral Lift` (still the best choice on slow-Z Klipper) and `retract_lift_enforce: Top Only` (already limits hops to top surface travels).

**Reversibility**: if reduced hop height causes visible travel drag on top surfaces (especially with warm ironed PLA), bump back to 0.3 or 0.4.

**Source**: Our own [`printer/extruder.md`](../printer/extruder.md) § Z hop, Klipper motion timing maths.

### 58. `filament_retract_restart_extra`: `0.03` → `0.1` (Sunlu Matte filament profile)

**Why**: the visible dip + step on the traditional seam (inside of hole). 0.03 mm was an empirical tweak that's too conservative — community consensus for filling visible seam dips is 0.1-0.25 mm. Our 0.03 was ~5× below that range.

**What it does**: adds 0.1 mm of extra extrusion immediately after de-retract, filling the under-extruded region at the loop start that pressure advance compensation leaves behind. This specifically addresses the "dip" component of the visible seam.

**Risk**: too high → creates a bump instead of a dip. 0.1 mm is in the lower half of the community range, so tuning headroom remains (can go up to 0.15 if still visible, or back off to 0.08 if bumps appear).

**Does NOT fix the "step" component** of the seam — that's the one-layer-height Z transition which is physically unavoidable for traditional seams. Only scarf seams eliminate that.

**Source**: [Prusa forum — under-extrusion at seams](https://forum.prusa3d.com/forum/input-shaper-mini/under-extrusion-or-gaps-at-seams-with-input-shaper-settings-and-5-x-firmware/paged/2/) (community values 0.15-0.25), [Duet3D forum](https://forum.duet3d.com/topic/16378/understanding-slic3r-s-retract-and-pressure-advance), [OrcaSlicer Issue #7326 begna112 comment](https://github.com/SoftFever/OrcaSlicer/issues/7326) (negative values for opposite problem).

## Verification

All 4 changes applied and verified programmatically:
```
[OK] Process: max_volumetric_extrusion_rate_slope: 150
[OK] Printer:  wipe_distance: ['1']
[OK] Printer:  z_hop: ['0.2']
[OK] Filament: filament_retract_restart_extra: ['0.1']
```

## Settings NOT changed this pass

User accepted the "apply all" option (a), not the isolate-variables option (b). If the next test print shows some changes helping and others hurting, we'll need a second pass with isolated variables.

Things we deliberately did NOT touch:
- `outer_wall_speed: 100` — could drop to 75 but user wants to test the other changes first
- `outer_wall_line_width: 105%` — Adam L's 150% would help seams but hurt dimensional accuracy
- `pressure_advance: 0.045` — may have drifted since shaper retune but re-calibration is a separate workflow
- `seam_slope_type: external` — still external for continued side-by-side testing; can be switched back to `all` for production later

## What to look for in the next test print (re-printing the same annulus)

| Change | Expected improvement | How to tell |
|---|---|---|
| ERS 300 → 150 | Scarf slope appears as single smooth gradient, not 20 bands | Visual inspection of outer wall scarf region under raking light |
| wipe_distance 2 → 1 | Ironing drag marks become shorter (~50% reduction) | Ironed top surface has fewer / shorter streaks |
| z_hop 0.4 → 0.2 | Print time reduction | Compare print duration to previous print (should be ~15 sec shorter on this test piece, ~13% on larger prints) |
| restart_extra 0.03 → 0.1 | Visible seam dip on hole interior reduced | Fingernail test — dip should be shallower. Visual: less contrast on the seam line |

## Potential new problems to watch for

| Change | Possible regression | What it would look like |
|---|---|---|
| ERS 150 | Overhang flow transitions less smooth | Fuzzy or inconsistent overhang areas |
| wipe_distance 1 | Slightly more blob transfer on non-ironing retracts | Tiny blobs on top of walls (rare) |
| z_hop 0.2 | Travel drag on ironed surface | New drag marks on top surfaces |
| restart_extra 0.1 | Bump replacing dip | Raised bump at seam location instead of dip |

**If any of these appear, the next pass (option b) will isolate which change caused it.**

## Rollback

```bash
# Undo all 4 test-print-tune changes (restore pre-tune state)
cp orca_slicer/backups/2026-04-11-test-print-tune/"ZeroG Mercury One.json" "/Users/user/Library/Application Support/OrcaSlicer/user/default/machine/"
cp orca_slicer/backups/2026-04-11-test-print-tune/"0.20mm Standard @MyKlipper - PLA.json" "/Users/user/Library/Application Support/OrcaSlicer/user/default/process/"
cp orca_slicer/backups/2026-04-11-test-print-tune/"Sunlu PLA Matte @MyKlipper 0.4 nozzle.json" "/Users/user/Library/Application Support/OrcaSlicer/user/default/filament/base/"
```
