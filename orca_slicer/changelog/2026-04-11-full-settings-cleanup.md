# OrcaSlicer settings cleanup — 2026-04-11

> Full settings cleanup across the printer, process, and filament profiles, based on the per-setting walkthrough completed on 2026-04-11 using the research documented in `orca_slicer/printer/`, `orca_slicer/process/`, and `orca_slicer/filament/`.
>
> **Scope of this change**: printer + process + filament (Sunlu PLA Matte). **Seam and ironing settings were deliberately skipped per user request** — they will be reviewed separately.
>
> **Prerequisite met**: OrcaSlicer was fully closed before any JSON edit (verified via `pgrep`).

---

## Files backed up

All three active profiles were copied to `orca_slicer/backups/2026-04-11-full-settings-cleanup/` before any changes:

1. `ZeroG Mercury One.json` — printer profile (from `~/Library/Application Support/OrcaSlicer/user/default/machine/`)
2. `0.20mm Standard @MyKlipper - PLA.json` — process profile (from `~/Library/Application Support/OrcaSlicer/user/default/process/`)
3. `Sunlu PLA Matte @MyKlipper 0.4 nozzle.json` — filament profile (from `~/Library/Application Support/OrcaSlicer/user/default/filament/base/`)

**Rollback**: if any of these changes causes a problem, copy the backup back over the live profile and restart OrcaSlicer.

---

## Summary

| File | Changes |
|---|---|
| `ZeroG Mercury One.json` | 16 changes |
| `0.20mm Standard @MyKlipper - PLA.json` | 16 changes |
| `Sunlu PLA Matte @MyKlipper 0.4 nozzle.json` | 5 changes |
| **Total** | **37 changes** |

---

## Printer profile changes — `ZeroG Mercury One.json`

### Group A — Basic Information tab

#### 1. `printable_area`: `260×240` → `260×245`

**Research**: [`orca_slicer/printer/basic_information.md`](../printer/basic_information.md) § Printable space.

**Why**: Klipper `[stepper_y] position_max` is **245 mm**, confirmed from live `~/printer_data/config/printer.cfg`. The OrcaSlicer profile had 240 mm, wasting 5 mm of physical bed area. The slicer was refusing to place objects in the rear-most 5 mm of a bed that could physically reach them.

**Impact**: Recovers 5 mm × 260 mm = 1 300 mm² of usable bed.

**Risk**: None. Klipper enforces `position_max` regardless of what the slicer emits, so the worst case was the existing behaviour (5 mm of bed unused). This fix makes the slicer's planning match physical reality.

---

#### 2. `printable_height`: (inherited 250 mm) → `258`

**Research**: [`orca_slicer/printer/basic_information.md`](../printer/basic_information.md) § Printable space.

**Why**: Klipper `[stepper_z] position_max` is **258 mm**. Our Orca profile inherited 250 from the `MyKlipper 0.4 nozzle` parent, wasting 8 mm of vertical envelope.

**Impact**: Recovers 8 mm of Z height on tall prints.

**Risk**: None. Same reasoning as above — Klipper enforces the real limit regardless.

---

#### 3. `printer_structure`: (inherited `Undefined`) → `CoreXY`

**Research**: [`orca_slicer/printer/basic_information.md`](../printer/basic_information.md) § Advanced, footnote `[^printer-structure]`.

**Why**: Our hardware is CoreXY (Mercury One.1 + Nebula Pro frame). Currently set to `Undefined`, which means OrcaSlicer can't apply CoreXY-specific gcode planning optimisations (if any). Accurate metadata is strictly better than placeholder metadata.

**Impact**: May unlock CoreXY-specific planning behaviours in future Orca versions. No immediate behavioural change.

**Risk**: None. Wrong metadata is only worse than accurate metadata.

---

#### 4. `thumbnails`: `"48x48/PNG, 300x300/PNG"` → `"32x32/PNG, 400x300/PNG"`

**Research**: [`orca_slicer/printer/basic_information.md`](../printer/basic_information.md) § Advanced, footnote `[^thumbnails]`.

**Why**: Mainsail's documented thumbnail sizes are 32×32 (file list) + 400×300 (print page). Our 48×48 + 300×300 works but is non-standard — Mainsail will scale/letterbox them.

**Impact**: Thumbnails match Mainsail's native display dimensions exactly, no scaling artefacts.

**Risk**: None. Both sizes are functional; this just matches the documented UI specs.

**Reference**: <https://docs.mainsail.xyz/slicers/orcaslicer/>

---

#### 5. `bed_mesh_min`: (default `-99999, -99999`) → `5, 36`

**Research**: [`orca_slicer/printer/basic_information.md`](../printer/basic_information.md) § Adaptive bed mesh. Live Klipper config confirmed via `curl printer.cfg` during the 2026-04-11 walkthrough.

**Why**: Klipper `[bed_mesh] mesh_min: 5, 36` is the actual probe-accessible origin. The slicer-side default `-99999` means "no constraint" — Klipper enforces the real bounds anyway, but making the slicer aware of them means adaptive mesh planning is honest.

**Impact**: Slicer knows the real mesh bounds when planning adaptive mesh probe points.

**Risk**: None. The slicer's constraint matches Klipper's actual constraint.

---

#### 6. `bed_mesh_max`: (default `99999, 99999`) → `255, 240`

**Research**: Same as above. Klipper `[bed_mesh] mesh_max: 255, 240`.

**Why / Impact / Risk**: Same as above, for the upper bound.

---

### Group B — Motion Ability tab (the 4× acceleration mismatch — highest-impact group)

#### 7. `machine_max_speed_z`: `["20", "12"]` → `["5", "5"]`

**Research**: [`orca_slicer/printer/motion_ability.md`](../printer/motion_ability.md) § Maximum speed, footnote `[^klipper-printer-cfg]`.

**Why**: Klipper `max_z_velocity: 5` — set deliberately for the Beacon-equipped 3-Z drivetrain. OrcaSlicer thought Z could do 20 mm/s (4× reality), so it planned Z-hop timing and layer change moves for nonexistent headroom. With the real 5 mm/s limit, a 0.4 mm Z hop takes 80 ms which is noticeable and is exactly why our extruder-tab `z_hop_types: Spiral Lift` override exists — to eliminate that pause. If the slicer plans with wrong Z speed, the timing model is broken.

**Impact**: Slicer plans Z moves with the actual speed limit. Time estimates for layer-change pauses become accurate.

**Risk**: None. Klipper enforces 5 mm/s regardless.

---

#### 8. `machine_max_acceleration_x`: `["40000", "20000"]` → `["10000", "5000"]`

**Research**: [`orca_slicer/printer/motion_ability.md`](../printer/motion_ability.md) § Maximum acceleration — "the single most impactful setting on this tab".

**Why**: Klipper `max_accel: 10000` is the real ceiling, enforced at execution time. The slicer had 40 000 — 4× too high. Slicer consequences of the 4× mismatch:

1. **Wrong velocity ramping at corners**: slicer thinks it can decelerate from 600 mm/s to 100 mm/s in ~12 mm, but with real 10 000 accel it actually needs ~50 mm. Tight features shorter than 50 mm get gcode planned for impossible deceleration, Klipper has to slow further, producing visible velocity discontinuities.
2. **Wrong time estimates**: slicer's time predictions are optimistic by 1.5-3×.
3. **Wrong pressure advance compensation timing**: PA calculations assume the wrong accel curve.
4. **Wrong junction deviation / corner velocity**: corners the slicer thinks the printer takes cleanly actually need more deceleration.

Klipper Estimator post-processes the gcode and corrects time estimates, but it can't fix velocity planning that's already baked in.

**Why `5000` for the "silent" value too?**: Klipper doesn't have a silent mode, so the second value is irrelevant. We set it to 5000 to keep the conventional "silent ≈ 50% of normal" ratio.

**Impact**: Slicer velocity planning becomes honest. Tight-feature deceleration planning matches reality.

**Risk**: None. Klipper enforces 10 000 regardless of what the slicer emits. The change is slicer-planning side only.

---

#### 9. `machine_max_acceleration_y`: `["40000", "20000"]` → `["10000", "5000"]`

**Research / Why / Impact / Risk**: Same as X above. 4× mismatch with Klipper.

---

#### 10. `machine_max_acceleration_z`: (inherited `["500", "200"]`) → `["50", "50"]`

**Research**: [`orca_slicer/printer/motion_ability.md`](../printer/motion_ability.md) § Maximum acceleration.

**Why**: Klipper `max_z_accel: 50` — 10× lower than the inherited slicer value. Reflects the real mechanical limit of the 3-Z drivetrain. The slicer's inherited 500 made Z-hop and layer change planning 10× optimistic.

**Impact**: Z acceleration planning matches reality.

**Risk**: None — Klipper enforces 50 regardless.

---

#### 11. `machine_max_acceleration_extruding`: `["40000", "20000"]` → `["10000", "5000"]`

**Research**: [`orca_slicer/printer/motion_ability.md`](../printer/motion_ability.md) § Maximum acceleration.

**Why**: Klipper uses the same `max_accel: 10000` ceiling for extruding moves. Slicer had 40 000. 4× too high — same consequences as X/Y above.

**Impact**: Printing-move deceleration planning matches reality.

**Risk**: None.

---

#### 12. `machine_max_acceleration_retracting`: `["40000", "5000"]` → `["10000", "5000"]`

**Research**: Same.

**Why**: Klipper uses `max_accel: 10000` for retraction-with-motion moves too (no separate retract accel). Slicer had 40 000 normal. 4× too high. Note: the "silent" value was already 5 000, only the "normal" is changing.

**Impact**: Retraction-with-motion planning matches reality.

**Risk**: None.

---

#### 13. `machine_max_acceleration_travel`: (inherited `["20000", "20000"]`) → `["10000", "5000"]`

**Research**: Same.

**Why**: Klipper uses the same `max_accel: 10000` for travel and printing. Inherited 20 000 was 2× too high.

**Impact**: Travel move planning matches reality. Expect slightly more honest print time estimates (less optimistic).

**Risk**: None. Change is slicer-side planning only.

---

### Group C — Extruder tab (the Adam L "do not" combination)

#### 14. `retract_when_changing_layer`: `"0"` → `"1"`

**Research**: [`orca_slicer/printer/extruder.md`](../printer/extruder.md) § Critical finding flagged at the top. Cross-reference: [`orca_slicer/process/seam_diagnosis_field_notes.md`](../process/seam_diagnosis_field_notes.md).

**Why**: Adam L's "Better Seams" guide (the community reference for scarf seams) explicitly identifies the combination `retract_when_changing_layer: 0 + wipe: 0` as a failure mode causing seam gorges. The wall ends with full nozzle pressure, the next layer starts with pressure deficit after retraction, and the cumulative effect is visible seam artefacts that look gorge-like. **Our profile had both off** — the exact avoid-list combination. Klipper's base profile default is **on**; this mismatch was probably introduced by an earlier copy-paste.

**Impact**: Layer transitions now retract first, eliminating ooze during the layer change. Combined with `wipe: 1` (below), fixes the seam pressure-relief failure mode.

**Risk**: Minor — may cause a slight print-time increase (extra retract per layer). Mitigation: retraction length is already tuned (1.0 mm empirically). If stringing appears, that's an indication the retraction length needs retuning, not that this change was wrong.

**Reference**: <https://www.printables.com/model/783313-better-seams-an-orca-slicer-guide-to-using-scarf-s>

---

#### 15. `wipe`: `"0"` → `"1"`

**Research**: Same as above. This is the other half of Adam L's "do not" combination.

**Why**: Without wipe-while-retracting, the nozzle pulls back from the wall end with full residual pressure, depositing a blob. The wipe move drags the nozzle inward during retraction, bleeding off pressure cleanly against the interior of the part.

**Impact**: Walls end cleanly. Pairs with `wipe_distance: 2 mm` (next change) to give the wipe something to work with.

**Risk**: Minor — may cause a slight print-time increase from the wipe moves. Same stringing-mitigation logic as above.

---

#### 16. `wipe_distance`: (inherited, effectively 2 mm but not explicit) → explicit `["2"]` mm

**Research**: [`orca_slicer/printer/extruder.md`](../printer/extruder.md) § Retraction, footnote `[^wipe-distance-footgun]`.

**Why**: Previously inherited from the Klipper-common parent. Making it explicit means the value is visible in the profile JSON and survives any future parent-profile changes. 2 mm is the community-standard wipe distance. **Note**: this setting does nothing unless `wipe: 1` — it's the classic "footgun" where users set wipe distance and wonder why nothing wipes. Fixed by #15 above.

**Impact**: Makes the wipe distance explicit and version-controlled alongside other overrides.

**Risk**: None. The effective value is unchanged.

**Reference**: The `seam_diagnosis_field_notes.md` describes the "gorge through the wall" failure mode where a user thought wipes were happening but the gcode contained zero wipe moves — wipe distance was set to 2 mm but `wipe` was off. This fix avoids that footgun.

---

## Process profile changes — `0.20mm Standard @MyKlipper - PLA.json`

### Group D — Quality tab

#### 17. `outer_wall_line_width`: (inherited 100%) → `"105%"`

**Research**: [`orca_slicer/process/quality_PLA.md`](../process/quality_PLA.md) § Line width, footnotes `[^vzbot-bundled]` and `[^line-width-bambu]`.

**Why**: Community consensus (VzBot bundled profile, Bambu Lab wiki) is 105-120% for outer wall. OrcaSlicer wiki's conservative 100% recommendation prioritises pure dimensional accuracy; the community consensus prioritises surface quality + wall bonding. The 0.02 mm difference (0.4 mm → 0.42 mm) is well below our XY backlash, so dimensional accuracy is effectively unchanged.

**Impact**: Slightly cleaner outer-wall surface; better inner-outer wall bonding.

**Risk**: None. VzBot bundled uses this exact value (0.42 mm) as their profile default.

---

#### 18. `inner_wall_line_width`: (inherited 110%) → `"115%"`

**Research**: Same source.

**Why**: Community consensus is ≥112-120% for inner wall to enhance structural strength and bond to the outer wall. We went with 115% — middle of the community range, better wall bonding without affecting outer dimensions.

**Impact**: Slightly better inner-to-outer wall bond; stronger wall stack.

**Risk**: None — inner wall line width doesn't affect visible dimensions.

---

#### 19. `enable_arc_fitting`: (inherited `false`) → `"1"`

**Research**: [`orca_slicer/process/quality_PLA.md`](../process/quality_PLA.md) § Precision, footnote `[^vzbot-bundled]`.

**Why**: Replaces short G1 line sequences with G2/G3 arc commands — smaller gcode files, smoother curves, better motion planning. The VzBot bundled profile ships with this enabled. Klipper has supported G2/G3 arcs since 2020 and our `printer.cfg` has `[gcode_arcs]` enabled. One community source claims "Klipper users should disable arc-related features" — that source is wrong and predates `[gcode_arcs]` support.

**Impact**: Smaller gcode files on curved features; smoother curved-surface quality.

**Risk**: None. Klipper handles arcs natively.

**Reference**: <https://www.klipper3d.org/G-Codes.html#g2-g3-controlled-arc-move>

---

#### 20. `xy_hole_compensation`: (inherited 0) → `"0.075"` mm

**Research**: [`orca_slicer/process/quality_PLA.md`](../process/quality_PLA.md) § Precision, footnote `[^xy-hole]`.

**Why**: VzBot bundled value. Compensates for the slicer's tendency to under-size circular holes slightly due to how it plans curved extrusion paths. 0.075 mm is the standard community value.

**Caveat (documented)**: GitHub Issue #8011 reports that this can cause a visible ridge artefact on holes that have a *partial opening to the outside* of the part (e.g., a slot that breaks through the side wall). For most prints this isn't an issue. If we hit it, we can override per-print or per-object.

**Impact**: Circular holes come out closer to modelled dimensions.

**Risk**: Minor — the partial-opening artefact case. Most prints don't have that geometry.

**Reference**: <https://github.com/SoftFever/OrcaSlicer/issues/8011>

---

#### 21. `elefant_foot_compensation`: (inherited 0.15) → `"0.1"` mm

**Research**: [`orca_slicer/process/quality_PLA.md`](../process/quality_PLA.md) § Precision, footnote `[^elephant-foot]`.

**Why**: VzBot bundled value and Orca wiki starting-point recommendation. Our inherited 0.15 is at the conservative end of the 0-1 mm range. Tuned printers with accurate first-layer Z (which we have via the Beacon, σ ≈ 60 nm) typically need *less* compensation, not more — the elephant-foot effect is mostly caused by first-layer over-squish, which we don't have.

**Impact**: More accurate first-layer dimensions for parts that mate or need precise base fitment.

**Risk**: If we somehow do have first-layer over-squish that was being masked by the higher compensation value, first-layer edges might get slightly fat. Unlikely given the Beacon's accuracy. Mitigation if needed: drop bed temp 5 °C.

---

#### 22. `precise_outer_wall`: (inherited `false`) → `"1"`

**Research**: [`orca_slicer/process/quality_PLA.md`](../process/quality_PLA.md) § Precision, footnote `[^precise-wall]`.

**Why**: Sets outer-to-inner wall overlap to zero, preventing the inner wall from "pushing out" the outer wall. Explicitly recommended with our `inner-outer-inner` wall order. Strength of the part is unaffected (per the OrcaSlicer discussion thread).

**Impact**: Slightly more accurate outer-wall dimensions.

**Risk**: None specific to our setup. Note: one community source incorrectly claims Klipper users should disable this — same wrong source as for arc fitting.

---

#### 23. `only_one_wall_top`: (inherited `false`) → `"1"`

**Research**: [`orca_slicer/process/quality_PLA.md`](../process/quality_PLA.md) § Walls and surfaces.

**Why**: Uses a single perimeter on the topmost layers, leaving more room for the top-surface solid infill. This gives the ironing pass (which is enabled in our profile) a larger area to flatten, producing a better final top surface.

**Impact**: Better top-surface quality on prints with visible flat tops.

**Risk**: Minor — can produce a slight "lip" where the single top wall meets the ironed infill on very narrow top surfaces. Mitigated by the `min_width_top_surface` threshold which only applies the feature on surfaces wide enough to hide the lip.

---

#### 24. `ensure_vertical_shell_thickness`: (inherited `enabled`) → explicit `"moderate"`

**Research**: [`orca_slicer/process/quality_PLA.md`](../process/quality_PLA.md) § Walls and surfaces, footnote `[^vertical-shell]`.

**Why**: The legacy `enabled` value may map to the modern `all` mode. Multiple GitHub issues report that `all` causes a visible "hull line" artefact on prints (notably Benchies and curved surfaces). `moderate` is the safer middle ground — it prevents the worst shell gaps without introducing the hull line.

**Impact**: Avoids the hull-line artefact on curved-surface prints.

**Risk**: Very minor — on some edge-case geometries, sloped features might show slightly more shell gapping than with `all`. Not enough to matter for normal PLA quality prints.

---

#### 25. `wall_generator`: (inherited `classic`) → `"arachne"`

**Research**: [`orca_slicer/process/quality_PLA.md`](../process/quality_PLA.md) § Wall generator, footnote `[^wall-gen-wiki]`.

**Why**: Arachne is the modern variable-width wall algorithm (originating in Cura, adopted into PrusaSlicer and OrcaSlicer). Handles thin features, text, and narrow gaps far better than classic by dynamically varying wall width instead of gap-filling. Slight slicing time cost for real quality benefit on detailed prints.

**Impact**: Cleaner thin-feature reproduction (text, fine details, narrow slots).

**Risk**: None known for general PLA prints. Very minor slicing-time increase.

**Reference**: <https://github.com/SoftFever/OrcaSlicer/wiki/quality_settings_wall_generator>

---

#### 26. `bridge_flow_ratio`: (inherited 1.0) → `"0.95"`

**Research**: [`orca_slicer/process/quality_PLA.md`](../process/quality_PLA.md) § Bridging, footnote `[^bridge-flow]`.

**Why**: Community consensus and VzBot bundled value. Reducing bridge flow slightly prevents sag on unsupported spans as gravity pulls on the molten strand before it cools.

**Impact**: Cleaner bridge undersides.

**Risk**: None.

**Reference**: <https://github.com/SoftFever/OrcaSlicer/wiki/quality_settings_bridging>

---

#### 27. `internal_bridge_flow_ratio`: (inherited 1.5) → `"1.0"`

**Research**: [`orca_slicer/process/quality_PLA.md`](../process/quality_PLA.md) § Bridging, footnote `[^internal-bridge]`.

**Why**: Default 1.5 over-flows internal bridges (the layer that closes a sparse infill cavity) to ensure coverage. Multiple GitHub issues report this causes lumps that protrude above the current layer, **sometimes causing nozzle crashes**. Modern community consensus is 1.0 (or 0.9 for severe cases). The actual flow is multiplied by the regular bridge flow ratio, so 1.0 × 0.95 = 0.95 effective.

**Impact**: Eliminates the risk of internal bridge lumps hitting the nozzle. Cleaner solid-infill closure.

**Risk**: None — the change is *towards* a safer default.

---

### Group E — Speed tab

#### 28. `inner_wall_speed`: `"180"` → `"200"`

**Research**: [`orca_slicer/process/speed_PLA.md`](../process/speed_PLA.md) § Other layer speed.

**Why**: Klipper common base default is 200. Our 180 was conservative. Inner walls are hidden so the speed only affects print time, not visible quality. 200 mm/s is well within the Rapido 2 HF volumetric flow envelope.

**Impact**: Saves a few minutes per print on models with lots of inner walls.

**Risk**: None. Inner walls don't show on visible surfaces.

---

#### 29. `top_surface_speed`: `"80"` → `"100"`

**Research**: Same doc § Other layer speed.

**Why**: Klipper common base default is 100 mm/s. Our 80 was very conservative. With ironing enabled (which re-flattens the top surface afterwards), the top-surface-infill speed barely matters — the ironing pass is what determines final surface quality.

**Impact**: Slightly faster top-surface printing. No quality loss because of ironing.

**Risk**: None with ironing enabled. If ironing ever gets disabled and we still print at 100 mm/s top surface, the quality might drop slightly — worth remembering.

---

#### 30. `travel_speed`: `"500"` → `"600"`

**Research**: Same doc § Travel speed.

**Why**: Our printer's `max_velocity: 600` is the actual mechanical ceiling. Anything above 600 just clamps at 600. Setting it to exactly 600 makes the slicer plan travel moves at the true ceiling.

**Impact**: Marginally faster travel moves. Total savings depend on travel volume in a given print.

**Risk**: None. Klipper enforces 600 anyway.

---

### Group F — Others tab

#### 31. `label_objects`: (inherited `false`) → `"1"`

**Research**: [`orca_slicer/process/others_PLA.md`](../process/others_PLA.md) § G-code output.

**Why**: Adds object labels in the gcode as comments so Klipper's `exclude_object` feature can identify and cancel individual objects mid-print. Our Klipper config already has `exclude_object: 1` enabled — but without `label_objects: true` in the slicer, there's nothing to exclude. This change activates the existing but unused Klipper capability.

**Impact**: Enables mid-print object cancellation via Mainsail / Fluidd's object-cancel UI.

**Risk**: None. Adds descriptive comments to gcode, no behavioural change.

**Reference**: <https://github.com/SoftFever/OrcaSlicer/wiki/others_settings_others>

---

#### 32. `post_process`: (empty) → `["/Users/user/klipper_estimator --config_file /Users/user/klipper_estimator.cfg post-process"]`

**Research**: [`orca_slicer/process/others_PLA.md`](../process/others_PLA.md) § Post-processing scripts. Also previously set up in the session where klipper_estimator was installed.

**Why**: Klipper Estimator post-processes the gcode after OrcaSlicer emits it but before Klipper executes it, recalculating print time estimates using the real Klipper motion planner. Without this, OrcaSlicer's time estimates are optimistic — typically by 1.5-3× on motion-heavy prints. With it, estimates are accurate within ±5 s over a multi-hour print.

**Important**: uses `--config_file` (static) not `--config_moonraker_url` (live). Memory note `klipper_estimator_use_static_config` explicitly says the Moonraker URL approach is "flaky as fuck" and we should always use the static cfg at `/Users/user/klipper_estimator.cfg`. Re-dump only when motion limits change — and since we're about to change motion limits (Group B above), the klipper_estimator.cfg should be re-dumped after this settings cleanup to match.

**Impact**: Accurate print time estimates in the slicer UI and in Mainsail's print queue.

**Risk**: The post-processing adds ~2-5 seconds per slice. Negligible.

**Follow-up needed**: After this settings cleanup is applied and `printer.cfg` motion limits are unchanged (they're already 10000/600), the klipper_estimator.cfg should still be correct because the Klipper-side values haven't changed — only the slicer-side values are being brought into alignment. **No re-dump needed.**

---

## Filament profile changes — `Sunlu PLA Matte @MyKlipper 0.4 nozzle.json`

### Group G — Cosmetic / metadata fixes

#### 33. `filament_vendor`: `["eSUN"]` → `["SUNLU"]`

**Research**: [`orca_slicer/filament/sunlu_pla_matte.md`](../filament/sunlu_pla_matte.md) § Findings flagged at the top.

**Why**: 🐛 **Bug**. Profile was cloned from an eSUN profile and the vendor field was never updated. The profile name says "Sunlu PLA Matte" but the vendor field says "eSUN". Purely cosmetic but confusing for anyone reading the profile.

**Impact**: Profile metadata is internally consistent. OrcaSlicer may also group/filter profiles by vendor in some UI views.

**Risk**: None.

---

#### 34. `filament_density`: `["1.23"]` → `["1.24"]`

**Research**: Same doc § Flow / material properties.

**Why**: Standard PLA density is 1.24 g/cm³. Matte PLA variants typically range 1.24-1.30 depending on filler loading. Our 1.23 was slightly below the standard PLA baseline. Affects spool-weight → filament-length conversion for cost tracking and runout detection only — **no slicing impact**.

**Impact**: Slightly more accurate filament-length-remaining estimates when using spool weight tracking.

**Risk**: None.

---

#### 35. `temperature_vitrification`: `["60"]` → `["55"]`

**Research**: Same doc § Temperature, footnote `[^sunlu-official]`.

**Why**: Matte PLA's actual glass transition temperature (Tg) is 50-55°C — slightly lower than standard PLA's ~60°C because of the amorphous-state filler additives. Our 60°C was optimistic. Used for slicer overheat warnings only — affects when the slicer warns that the chamber/environment temperature is approaching the filament's softening point. No slicing impact.

**Impact**: Slicer's "overheat warning" becomes honest.

**Risk**: None. Changes a warning threshold only.

---

### Group H — Z hop regression fix (overrides that silently undid printer-tab decisions)

#### 36. `filament_z_hop`: `["0.2"]` → **remove key** (inherit printer-tab 0.4)

**Research**: [`orca_slicer/filament/sunlu_pla_matte.md`](../filament/sunlu_pla_matte.md) § Z hop (filament-side overrides) — flagged section.

**Why**: This filament-level override halves the printer-tab Z hop (0.4 → 0.2 mm) whenever the Sunlu filament is loaded. Unclear if deliberate or a copy-paste artefact. Paired with #37 below (z_hop_types) which is clearly a silent regression.

**Impact**: Z hop behaviour for this filament now matches the printer tab (0.4 mm).

**Risk**: If the 0.2 mm override was a deliberate empirical tweak we've forgotten, we might see slightly more nozzle drag on large-top-surface prints with this filament. Mitigation: re-add the override if drag appears.

---

#### 37. `filament_z_hop_types`: `["Normal Lift"]` → **remove key** (inherit printer-tab `Spiral Lift`)

**Research**: Same. This is the clear regression.

**Why**: The extruder tab explicitly sets `z_hop_types: Spiral Lift` because our Klipper `max_z_velocity: 5 mm/s` makes vertical-lift pauses noticeable (~40 ms at 0.2 mm, ~80 ms at 0.4 mm). Spiral Lift corkscrews up while travelling — no pause. Normal Lift (this override) is "lift vertically then travel" — adds the pause back. This override silently undoes the extruder-tab decision whenever the Sunlu filament is loaded.

**Impact**: Z hop for this filament becomes Spiral Lift, matching the extruder-tab intent. ~40-80 ms saved per retraction.

**Risk**: None. Spiral Lift works with any filament — no filament-specific reason Normal Lift was needed.

**Reference**: See [`orca_slicer/printer/extruder.md`](../printer/extruder.md) § Retraction → "Z hop type".

---

## Deferred — NOT changed in this pass

### Seam settings
All seam-related settings across `quality_PLA.md` were deliberately skipped. The user wants to review scarf seams separately in a future pass.

### Ironing settings
All ironing-related settings across `quality_PLA.md`, `speed_PLA.md`, and `sunlu_pla_matte.md` were deliberately skipped. The user wants to review ironing separately.

### `filament_max_volumetric_speed: 24` (Sunlu PLA Matte)
Flagged as high (2× the bundled default 12 mm³/s for matte PLA). Matte PLA is typically filler-limited to 12-16 mm³/s. Needs a **Max Volumetric Speed calibration print** before changing. Guessing a lower number would just replace one untested value with another untested value.

**Action needed**: run OrcaSlicer's Max Volumetric Speed calibration test with this filament on the Rapido 2 HF + Orbiter 2.5 + hardened steel 0.4 mm nozzle setup. If 24 mm³/s passes cleanly, leave it. If it fails around 15-18, drop to the tested ceiling.

### Empirically-calibrated values left alone
- `retraction_length: 1.0` (printer tab) — calibrated
- `retraction_speed: 120` and `deretraction_speed: 120` (printer tab) — calibrated for Orbiter
- `pressure_advance: 0.045` (Sunlu filament) — calibrated, matches field notes
- `filament_flow_ratio: 0.98` (Sunlu filament) — calibrated
- All fan/cooling overrides on Sunlu profile — empirically tuned for matte PLA

---

## Discrepancies found during application (not changed)

While reading the live profile JSON for editing, I noticed several settings that are in the actual profile but aren't covered (or are covered differently) in the research docs. **These are noted for a future documentation update — not changed in this pass.**

### In `0.20mm Standard @MyKlipper - PLA.json`:

1. **`seam_position: "aligned_back"`** — The `quality_PLA.md` doc still says our current value is `"nearest"` and recommends changing to `aligned_back`. The actual profile already has `aligned_back`. **Doc is stale — the change was already made in practice.** Update the doc during the seam review pass.

2. **`sparse_infill_density: "7%"`** — The `quality_PLA.md` Strength section (via cross-reference to `strength_PLA.md`) says our value is `15%` (the default). The actual profile has `7%`. That's very low — halfway between "decorative" (10%) and "below minimum" (<10%). Worth understanding *why* 7% was chosen. Not changed in this pass since we didn't have a research backing for either value.

3. **`fuzzy_skin: "all"` and `fuzzy_skin_noise_type: "ridgedmulti"`** — The `others_PLA.md` doc says we leave fuzzy skin off (default `none`). The actual profile has `fuzzy_skin: "all"` which means **fuzzy skin is currently applied to every wall on every print**. That's a significant behavioural difference from what the doc describes. The `ridgedmulti` noise type is a specific OrcaSlicer option. **This looks like an experimental setting that was left on.** Not changed in this pass — flagged for user attention.

4. **`support_style: "organic"` and `support_type: "tree(auto)"`** — The process README says "we don't really need support" and `others_PLA.md` doesn't cover these. But the profile has organic tree supports configured. Not a problem (supports only activate when the user enables them per-model), but worth noting the config exists.

**Recommendation**: flag these to the user post-application. Especially #3 (`fuzzy_skin: all`) which is likely a surprise.

---

## Post-application verification plan

After applying all 37 changes:

1. Read each modified JSON file and confirm the changes took effect
2. Check JSON validity (no stray commas, proper quoting)
3. Open OrcaSlicer and verify the profiles load without errors
4. Slice a simple test model to confirm no setting combinations break the slicer
5. (Optional) Slice a dimensional test (calibration cube) to verify the precision changes don't cause unexpected issues

---

## Rollback procedure

If any change causes a problem:

```bash
# From the repo root:
cp orca_slicer/backups/2026-04-11-full-settings-cleanup/"ZeroG Mercury One.json" \
   "/Users/user/Library/Application Support/OrcaSlicer/user/default/machine/"

cp orca_slicer/backups/2026-04-11-full-settings-cleanup/"0.20mm Standard @MyKlipper - PLA.json" \
   "/Users/user/Library/Application Support/OrcaSlicer/user/default/process/"

cp orca_slicer/backups/2026-04-11-full-settings-cleanup/"Sunlu PLA Matte @MyKlipper 0.4 nozzle.json" \
   "/Users/user/Library/Application Support/OrcaSlicer/user/default/filament/base/"
```

Then restart OrcaSlicer.

Individual changes can be reverted by diffing the backup against the live file and cherry-picking the specific keys to revert.

---

## Post-application verification (2026-04-11, same day)

All 37 changes were applied and verified programmatically via a Python JSON load + key-by-key comparison. Result: **37/37 OK** — every change landed correctly with the expected value.

### Final JSON key counts

| File | Before | After | Delta | Notes |
|---|---|---|---|---|
| `ZeroG Mercury One.json` | 31 | 39 | +8 | 8 new keys added (inc. `printable_height`, `printer_structure`, `thumbnails`, `bed_mesh_min/max`, `machine_max_acceleration_z`, `machine_max_acceleration_travel`, `wipe_distance`). 8 existing keys modified in place. |
| `0.20mm Standard @MyKlipper - PLA.json` | 47 | 62 | +15 | 15 new keys added (all Quality and Others changes that were previously inherited). 3 existing keys modified in place (inner/top/travel speed). 1 new key `post_process` added. |
| `Sunlu PLA Matte @MyKlipper 0.4 nozzle.json` | 123 | 121 | -2 | 3 existing keys modified in place (vendor, density, vitrification). 2 keys removed (`filament_z_hop`, `filament_z_hop_types`) so they inherit from the printer tab. |

### Next steps for the user

1. **Open OrcaSlicer** and confirm the three profiles load without errors.
2. **Check the settings UI** — the 37 changes should all be visible in their respective tabs.
3. **Slice a test print** to confirm the slicer doesn't hit any unexpected setting combinations.
4. **Watch for the Adam L "do not" fix** — the first print with `wipe: 1 + retract_when_changing_layer: 1` should show noticeably better seams on normal (non-scarf) corners.
5. **Klipper Estimator** — the post-processing line should run automatically on every slice and update the time estimate. Watch the slicer's status bar during slicing; you should see "Running post-processing..." briefly.
6. **Open the fuzzy_skin discrepancy** — `fuzzy_skin: "all"` was found unexpectedly in the profile. If you want prints without fuzzy skin, this needs changing separately.

### Rollback still available

The backups at `orca_slicer/backups/2026-04-11-full-settings-cleanup/` are byte-identical copies of the pre-change profiles. Use the commands in the "Rollback procedure" section above to restore.

---

## Second pass — 2026-04-11 (same day) — discrepancy fixes

Following the first-pass verification, the user approved fixing the three discrepancies found while reading the live profile. These were settings that were in the actual JSON but either contradicted the research docs or were experimental leftovers.

### Backup for second pass

The post-first-pass state of the process profile was copied to `orca_slicer/backups/2026-04-11-discrepancy-fix/0.20mm Standard @MyKlipper - PLA.json` before the second pass. This is the **post-first-pass, pre-second-pass** state — use it to roll back *only* the second-pass changes while keeping the first-pass changes intact.

**The original pre-first-pass backup at `orca_slicer/backups/2026-04-11-full-settings-cleanup/` is still valid for a full rollback.**

### Second-pass changes (5 total)

#### 38. `fuzzy_skin`: `"all"` → `"none"`

**File**: `0.20mm Standard @MyKlipper - PLA.json`

**Research**: [`orca_slicer/process/others_PLA.md`](../process/others_PLA.md) § Special mode, footnote `[^fuzzy-skin]`.

**Why**: 🚨 The profile had `fuzzy_skin: "all"` which applies fuzzy-skin texture to **every wall on every print**. This contradicts the research doc which explicitly says `fuzzy_skin: none` is the correct default for "general PLA quality printing" because fuzzy skin is "the opposite of 'smooth quality finish'". The `all` setting was likely an experimental leftover from a specific decorative print that never got reverted.

Use cases for fuzzy skin (from the research doc):
- Decorative grip surfaces
- Hiding layer lines on visible surfaces
- Organic-looking finishes

For our quality-first PLA profile, none of these apply as a default. Users who want fuzzy skin for a specific model can enable it per-model in the slicer prepare view.

**Impact**: Walls are smooth again (no random texture perturbations). First print after this change should look visibly different — cleaner, smoother outer walls.

**Risk**: None. This restores the intended default behaviour.

**Reference**: <https://github.com/SoftFever/OrcaSlicer/wiki/others_settings_fuzzy_skin>

---

#### 39. `fuzzy_skin_noise_type`: `"ridgedmulti"` → **remove key**

**File**: `0.20mm Standard @MyKlipper - PLA.json`

**Why**: Only relevant when `fuzzy_skin` is enabled. With fuzzy_skin set to `none` (change #38), this field does nothing. Removing it keeps the profile clean.

**Impact**: None functional — the setting was already inert after #38.

**Risk**: None.

---

#### 40. `sparse_infill_density`: `"7%"` → `"15%"`

**File**: `0.20mm Standard @MyKlipper - PLA.json`

**Research**: [`orca_slicer/process/strength_PLA.md`](../process/strength_PLA.md) § Infill, footnote `[^infill]`. The doc recommends 15% as the standard "quality plus reasonable strength" choice, citing the OrcaSlicer Infill Wiki.

**Why**: The profile had `sparse_infill_density: "7%"` which is **below even the community's "decorative" threshold** (10%). The research doc explicitly states 15% is the standard default, and both 15% (functional) and 10% (decorative) are the community range — 7% doesn't match any documented recommendation.

**Context**: We have `wall_loops: 4` + `sparse_infill_pattern: gyroid` which provides structural strength through walls and efficient 3-axis infill distribution. A 7% gyroid infill is technically viable for decorative prints (gyroid is efficient at low densities), but for a general quality-first profile, 15% provides a much better strength safety margin without significant time/material cost.

**Why 15% specifically**: It's the documented default; going to 15% aligns our profile with the doc and the community consensus.

**Impact**: Prints will have slightly more internal infill. Marginal print-time increase. Significant strength improvement on functional prints (especially impact/compressive loads).

**Risk**: None.

**Reference**: <https://github.com/SoftFever/OrcaSlicer/wiki/strength_settings_infill>

---

#### 41. `support_style`: `"organic"` → **remove key** (inherit from parent)

**File**: `0.20mm Standard @MyKlipper - PLA.json`

**Why**: The process README says "we don't really need support" — supports aren't part of our normal workflow. These overrides (`support_style` and `support_type` — #42) were in the profile as experimental leftovers from a specific print that needed supports.

Removing the overrides means:
- When supports are NOT enabled per-model (our normal case), nothing changes
- When supports ARE enabled per-model (occasional), the slicer uses the parent profile's defaults, which are sensible starting points

**Impact**: Cleaner profile JSON — one less override for a feature we don't use by default.

**Risk**: If the user enables supports per-model in the future, they'll get the parent profile's default support style (likely `default` or whatever the common base provides) rather than `organic`. Both styles work; `organic` (Cura-style tree) is slightly better-looking for complex overhangs, but `default` linear supports work fine too.

**If it turns out we want organic trees as the default for supports**: re-add the override. Easy to revert.

---

#### 42. `support_type`: `"tree(auto)"` → **remove key** (inherit from parent)

**File**: `0.20mm Standard @MyKlipper - PLA.json`

**Why**: Same as #41. Paired override for the tree-support type. Removing both keeps the profile clean.

**Impact / Risk**: Same as #41.

---

### Second-pass verification

All 5 second-pass changes were applied and verified programmatically:

```
PROCESS: valid JSON, 59 keys
  fuzzy_skin: none (expected: none)                    ✓
  fuzzy_skin_noise_type: REMOVED (expected: REMOVED)   ✓
  sparse_infill_density: 15% (expected: 15%)           ✓
  support_style: REMOVED (expected: REMOVED)           ✓
  support_type: REMOVED (expected: REMOVED)            ✓
```

**Result: 5/5 OK**

### Key count change from second pass

| File | Before second pass | After second pass | Delta |
|---|---|---|---|
| `0.20mm Standard @MyKlipper - PLA.json` | 62 | 59 | -3 |

3 key removals (`fuzzy_skin_noise_type`, `support_style`, `support_type`) minus 2 in-place modifications (`fuzzy_skin`, `sparse_infill_density`) = net -3 keys.

### Grand total for 2026-04-11

**42 total changes** across the three profiles on this session:
- Printer profile: 16 changes
- Process profile: 21 changes (16 first pass + 5 second pass)
- Filament profile: 5 changes

### Rollback options

**Full rollback** (undo everything from today):
```bash
cp orca_slicer/backups/2026-04-11-full-settings-cleanup/"ZeroG Mercury One.json" "/Users/user/Library/Application Support/OrcaSlicer/user/default/machine/"
cp orca_slicer/backups/2026-04-11-full-settings-cleanup/"0.20mm Standard @MyKlipper - PLA.json" "/Users/user/Library/Application Support/OrcaSlicer/user/default/process/"
cp orca_slicer/backups/2026-04-11-full-settings-cleanup/"Sunlu PLA Matte @MyKlipper 0.4 nozzle.json" "/Users/user/Library/Application Support/OrcaSlicer/user/default/filament/base/"
```

**Partial rollback** (undo ONLY the second pass, keep first-pass changes):
```bash
cp orca_slicer/backups/2026-04-11-discrepancy-fix/"0.20mm Standard @MyKlipper - PLA.json" "/Users/user/Library/Application Support/OrcaSlicer/user/default/process/"
```

Restart OrcaSlicer after either rollback.

---

## References

The detailed research for each change lives in:

- [`orca_slicer/printer/basic_information.md`](../printer/basic_information.md)
- [`orca_slicer/printer/motion_ability.md`](../printer/motion_ability.md)
- [`orca_slicer/printer/extruder.md`](../printer/extruder.md)
- [`orca_slicer/process/quality_PLA.md`](../process/quality_PLA.md)
- [`orca_slicer/process/speed_PLA.md`](../process/speed_PLA.md)
- [`orca_slicer/process/others_PLA.md`](../process/others_PLA.md)
- [`orca_slicer/filament/sunlu_pla_matte.md`](../filament/sunlu_pla_matte.md)
- [`orca_slicer/process/seam_diagnosis_field_notes.md`](../process/seam_diagnosis_field_notes.md) (for the Adam L "do not" combination reasoning)

All footnote citations in those docs point to the authoritative external sources (OrcaSlicer wiki, Bambu Lab wiki, Prusa Knowledge Base, VzBot bundled profile, GitHub issues).
