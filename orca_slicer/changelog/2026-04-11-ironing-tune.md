# OrcaSlicer ironing tuning — 2026-04-11

> Dedicated changelog for the ironing settings review and tuning pass. Builds on the full settings cleanup earlier today (see [`2026-04-11-full-settings-cleanup.md`](2026-04-11-full-settings-cleanup.md)) which deliberately skipped ironing to do it as a focused review.
>
> **Scope**: PLA-optimal ironing configuration in the process profile `0.20mm Standard @MyKlipper - PLA.json`.
>
> **Research base**: [`orca_slicer/process/ironing_deep_dive.md`](../process/ironing_deep_dive.md) — 22 cited references covering every ironing parameter, interaction, calibration workflow, and failure mode. Plus additional 2026 community research performed during this tuning pass (documented in the per-change rationale below).

---

## Why a separate changelog

Ironing settings were excluded from the main cleanup pass because the user wanted them reviewed in depth using the dedicated deep-dive research document. This changelog captures that focused review.

## Prerequisite met

OrcaSlicer was fully closed before any JSON edit (verified via `pgrep`). The process profile is already in its post-full-cleanup, post-discrepancy-fix state — this tuning pass builds on those changes without touching any non-ironing settings.

## Backup

The current state of `0.20mm Standard @MyKlipper - PLA.json` (post-full-cleanup + post-discrepancy-fix) was copied to `orca_slicer/backups/2026-04-11-ironing-tune/` before any edits. That's the third backup of this file today, giving us three distinct rollback points:

1. `backups/2026-04-11-full-settings-cleanup/` — pre-ANY-changes state (full original)
2. `backups/2026-04-11-discrepancy-fix/` — post-first-pass, pre-second-pass state
3. `backups/2026-04-11-ironing-tune/` — post-second-pass, pre-ironing-tune state *(this backup)*

## Pre-tune state

The 6 ironing-related keys already set in the profile from earlier calibration work:

| Key | Value | Status vs deep-dive |
|---|---|---|
| `ironing_type` | `top` | ✅ Already correct — iron all top surfaces |
| `ironing_flow` | `12%` | ✅ Already correct — slightly above default to compensate for our 0.98 flow ratio |
| `ironing_spacing` | `0.1` | ✅ Already correct — tighter than default for quality (4× coverage on 0.4 mm nozzle) |
| `ironing_speed` | `40` mm/s | ✅ Already correct — deliberate, faster than 20-30 mm/s Prusa-traditional to avoid heat creep on our Rapido 2 HF + Orbiter 2.5 |
| `ironing_angle` | `90` | ✅ Already correct — absolute 90° for uniform finish |
| `ironing_angle_fixed` | `1` | ✅ Already correct — enables absolute angle mode |

The deep dive's PLA-calibrated table in section 8 says **our current values ARE the recommended values** — so this pass is about filling in the two remaining "optional refinement" gaps the deep dive identified, plus one related cleanup.

## Settings NOT being changed

| Key | Why kept |
|---|---|
| `ironing_flow: 12%` | Already matches the deep dive's PLA recommendation. Community research on matte-PLA-specific flows (Bambu Matte 20-25%, Overture Matte 15%) suggests a higher value for matte variants, but our 12% is validated for our setup. If we want matte-specific override, the right place is `filament_ironing_flow` on the filament profile, not here on the process profile. |
| `ironing_speed: 40 mm/s` | Deliberate choice documented in the deep dive. Faster than Prusa-traditional 20 mm/s, slower than matte-community 30-55 mm/s values, balanced specifically to reduce heat-creep risk on our hardware. |
| `ironing_spacing: 0.1 mm` | Already at the quality-end of the community range (0.08-0.15 mm). Tighter = more coverage = longer time. 0.1 gives 4× coverage on a 0.4 mm nozzle which is the quality sweet spot. |
| `ironing_angle: 90°` + `ironing_angle_fixed: 1` | Already set to the OrcaSlicer Issue #10834 recommendation — absolute 90° for uniform finish across all top surfaces, reduces tiger striping. |
| `ironing_type: top` | Irons all top surfaces (vs `topmost` or `solid`). Right choice for general-purpose PLA. |

---

## Changes in this pass (3 total)

### 43. Add `ironing_pattern`: (inherited Concentric) → `"rectilinear"`

**Research sources**:
- Primary: [`orca_slicer/process/ironing_deep_dive.md`](../process/ironing_deep_dive.md) § 4.2 and § 10
- Secondary (2026 additional research):
  - Bambu Lab Wiki on ironing patterns [^bambu-ironing-patterns]
  - OrcaSlicer Issue #3796 on Concentric layer-shift artefact [^orca-issue-3796]
  - Community pattern comparison discussions [^pattern-comparison]

**Why**: The deep dive lists this as an optional refinement, but the 2026 additional research surfaces a stronger argument for the change than the deep dive captured.

**The deep dive's position** (section 4.2): "Concentric is safe for all shapes. Rectilinear + fixed angle can give better results on large flat tops."

**New information from 2026 research**:

1. **Concentric has a "center dot" artefact** — from community comparison discussions: "Concentric may deliver better ironing results on the edges of the top surface, but tends to leave a small dot in the center of the model" [^pattern-comparison]. This is a visible quality regression Concentric introduces that Rectilinear doesn't have. The deep dive mentioned a small-top-surface layer-shift artefact (issue #3796) but didn't capture this center-dot issue.

2. **Rectilinear is more versatile**: "Rectilinear is the more widely used option, with strong versatility" [^pattern-comparison]. Concentric is specifically recommended for circular/organic shapes, not general-purpose printing.

3. **Pairs optimally with our fixed 90° angle**: the deep dive explicitly says "Rectilinear + fixed angle gives slightly better tiger-stripe reduction." We already have `ironing_angle_fixed: 1` + `ironing_angle: 90` — Rectilinear is the pattern choice that most benefits from this combo.

**Why Rectilinear specifically**:
- Our typical print targets (functional parts, enclosures, lids, brackets) have predominantly flat rectangular top surfaces
- Concentric's strengths are rounded/organic tops which are the minority of our use cases
- The center-dot and spiral-closing-point artefacts that affect Concentric are real quality regressions
- Rectilinear's path geometry pairs optimally with our existing fixed-angle setting

**Impact**:
- Top surfaces will be smoother on flat rectangular prints (no center dot)
- Slightly better tiger-stripe reduction paired with our 90° fixed angle
- Marginally faster ironing on angular shapes (more efficient coverage)
- Round-top prints (coasters, circular bases) will be slightly less optimal than Concentric would be, but still acceptable

**Risk**: For pure cylindrical-top prints, a Concentric spiral would have been marginally better. This is a known trade-off; can override per-print if printing a cylinder.

**Reference**: <https://wiki.bambulab.com/en/software/bambu-studio/parameter/ironing>

---

### 44. Add `ironing_inset`: (inherited ~0.21 mm default) → `"0.15"` mm

**Research sources**:
- Primary: [`orca_slicer/process/ironing_deep_dive.md`](../process/ironing_deep_dive.md) § 4.5 and § 10
- Secondary (2026 additional research):
  - OrcaSlicer Issue #5812 — Ironing Inset [^orca-issue-ironing-inset]
  - OrcaSlicer Discussion #2778 — Feature request context [^orca-discussion-ironing-inset]
  - OrcaSlicer Wiki on Inset behaviour [^orca-wiki-ironing]

**Why**: The deep dive § 4.5 table:

| Inset (mm) | Effect |
|---|---|
| **0** | Ironing runs all the way to the edge. Maximum coverage, maximum edge fuzz |
| **0.1-0.15** | Slight inset — invisible in most cases. Cleanest result |
| **0.2-0.25 mm** | Orca default. Small visible ring of un-ironed edge on some features |
| **0.5+ mm** | Heavy inset. Only useful for large flat tops with critical edge geometry. Creates a visible "frame" around the ironed area |

The current inherited default (~0.21 mm) is at the top of Orca's "small ring" range. The **quality-focused** community value is **0.15 mm** — close enough to the edge that the ring is invisible, far enough to prevent edge fuzz.

**New information from 2026 research** (confirming the deep dive): "Inset > 0: The nozzle stays slightly inside the boundary, producing crisper edges and better-defined corners" [^orca-wiki-ironing]. This confirms 0.15 as the right quality value — it's inset enough to give crisper edges, not so much that it leaves a visible un-ironed ring.

**Why 0.15 specifically (vs 0.1 or 0.2)**:
- **0.1 mm**: closer to the edge → more coverage, but increased risk of edge fuzz from the nozzle's ~0.4 mm flat underside physically pushing melted plastic past the edge
- **0.15 mm**: community "cleanest result" value (per deep dive § 4.5 table). Far enough from the edge that the nozzle doesn't push material over, close enough that no visible un-ironed ring forms
- **0.2 mm**: the current inherited default. Slightly visible un-ironed ring on some features. Safer against fuzz, slightly worse coverage

**Impact**:
- Crisper, better-defined edges on ironed top surfaces
- Smaller un-ironed "dead zone" at the perimeter (0.15 vs 0.21 = 0.06 mm less dead zone per edge)
- Slightly more visible ironing on thin top features where the inset was consuming too much of the surface

**Risk**: Very minor. At 0.15 mm, the nozzle is still inset enough that physical edge push-over is avoided. If edge fuzz appears on specific prints, can bump back to 0.2 mm per-print.

---

### 45. Add `top_surface_pattern`: (inherited `monotonicline`) → explicit `"monotonicline"`

**Research sources**:
- Primary: [`orca_slicer/process/ironing_deep_dive.md`](../process/ironing_deep_dive.md) § 5.2 "Interaction with other quality settings — Top surface pattern"
- Secondary: [`orca_slicer/process/strength_PLA.md`](../process/strength_PLA.md) § Top/bottom shells, footnote `[^patterns]`

**Why**: The deep dive § 5.2 states:

> **Recommended with ironing**: Monotonic or Monotonic Line. **Do not use Hilbert Curve, Gyroid, or other fancy patterns** as the top surface under ironing — they don't lay flat enough for the iron to fix.

The current inherited value IS `monotonicline` (from the `fdm_process_common` base), which is correct. **This change doesn't modify behaviour** — it locks the value into the process profile so it survives future parent-profile changes and makes the ironing-pattern pairing explicit.

**Why explicit matters**:
- If OrcaSlicer ever updates the parent profile's default to a different pattern (e.g. `monotonic` without the "line" suffix, or back to `rectilinear`), we keep the ironing-optimal choice
- Self-documenting: anyone reading the profile sees the explicit pairing `top_surface_pattern: monotonicline` + ironing, and understands the relationship
- Matches the pattern of the other explicit settings we added in earlier passes

**monotonicline vs monotonic**: Both are valid. `monotonicline` is slightly faster because it doesn't try to start each line from a clean wall side. With ironing re-flattening the result, either works. Keeping `monotonicline` because that's what was inherited and working.

**Impact**: None (the inherited value is already `monotonicline`).

**Risk**: None.

---

## What about matte-specific ironing flow?

The 2026 additional research surfaced community values for matte PLA specifically:
- Bambu Matte PLA Black: Speed 30, Flow 25% [^bambu-matte-ironing]
- Bambu Matte PLA Grey: Speed 30, Flow 20% [^bambu-matte-ironing]
- Overture Matte PLA: Speed 55, Flow 15% [^bambu-matte-ironing]

These are **higher flows** than our process profile's 12%. The reasoning is that matte PLA's filler particles require more fresh material during ironing to fully even out the surface — the 10-15% that works for standard PLA leaves voids on matte surfaces.

**This suggests our Sunlu PLA Matte might benefit from a higher ironing flow** (say 15-20%) specifically for that filament, while keeping the process profile at 12% for generic PLA.

**Why we're NOT changing this right now**:

1. **Process profile scope**: the process profile is meant to be the generic PLA baseline. Matte-specific overrides belong on the filament profile via `filament_ironing_flow`.
2. **Calibration prerequisite**: per the deep dive § 6 calibration workflow, ironing flow should be validated with a calibration print before being changed. Guessing 15% or 20% without a test is just replacing one untested value with another.
3. **Our current 12% is working**: no evidence it's wrong. If top-surface quality is fine on matte PLA prints, there's no reason to change.
4. **Bambu values may not translate**: those settings are calibrated on Bambu hardware with different pressure advance tuning. Our Orbiter 2.5 + Rapido 2 HF + calibrated PA may behave differently.

**Action for later (not this changelog)**: print an ironing calibration model from MakerWorld with our Sunlu PLA Matte, find the true sweet spot, and set `filament_ironing_flow` on the Sunlu profile if the ideal matte value differs from 12%.

---

## Application order

1. ✅ Verify OrcaSlicer not running (done)
2. ✅ Check current ironing state (done, 6 keys set, 3 keys missing)
3. ✅ Back up current profile to `backups/2026-04-11-ironing-tune/` (done)
4. ⏳ Create this changelog file (you're reading it)
5. ⏳ Apply 3 edits via Edit tool
6. ⏳ Programmatic verification via Python JSON load
7. ⏳ Update the deep-dive doc to reflect the new explicit values
8. ⏳ Update the memory system if this changes future recommendations (probably not needed — deep dive already has the recommendations)

## Post-application verification plan

After applying all 3 changes:

1. Read the modified JSON file and confirm the 3 new keys are present with correct values
2. Check JSON validity (no stray commas, proper quoting)
3. Confirm no OTHER ironing settings were accidentally modified
4. Open OrcaSlicer and verify the profile loads without errors (user task)
5. Slice a test print with a flat top surface (~40×40 mm) to verify Rectilinear + fixed 90° produces clean results
6. Compare to pre-change behaviour if you have a before print to compare against

## Rollback procedure

```bash
# Undo ONLY the ironing tuning changes (keep the two earlier passes)
cp orca_slicer/backups/2026-04-11-ironing-tune/"0.20mm Standard @MyKlipper - PLA.json" \
   "/Users/user/Library/Application Support/OrcaSlicer/user/default/process/"
```

Or for a **full day rollback** to before any 2026-04-11 changes:

```bash
cp orca_slicer/backups/2026-04-11-full-settings-cleanup/"0.20mm Standard @MyKlipper - PLA.json" \
   "/Users/user/Library/Application Support/OrcaSlicer/user/default/process/"
```

---

## References

### Deep dive document references

All cited in [`orca_slicer/process/ironing_deep_dive.md`](../process/ironing_deep_dive.md), section 11. Key ones for this tuning pass:

- OrcaSlicer Wiki § Ironing Settings
- Bambu Lab Wiki § Ironing (pattern comparison)
- Prusa Knowledge Base § Ironing (material-specific)
- OrcaSlicer Issue #10834 (fixed angle recommendation)
- OrcaSlicer Issue #3796 (Concentric layer-shift on small tops)
- OrcaSlicer Issue #5411 (calibration workflow)
- OrcaSlicer Discussion #2778 (inset parameter rationale)

### Additional 2026 research used for this changelog

[^bambu-ironing-patterns]: Bambu Lab Wiki — Ironing parameter page, pattern descriptions. Confirms Rectilinear as the "more widely used, versatile" option vs Concentric's shape-specific strengths. <https://wiki.bambulab.com/en/software/bambu-studio/parameter/ironing>

[^orca-issue-3796]: OrcaSlicer Issue #3796 — "Layer shift when using concentric top surface pattern". Documents the small-top-surface closing-point artefact. Workaround: use rectilinear. <https://github.com/OrcaSlicer/OrcaSlicer/issues/3796>

[^pattern-comparison]: Community pattern comparison summary from 2026 search: "Concentric may deliver better ironing results on the edges of the top surface, but tends to leave a small dot in the center of the model. Rectilinear is the more widely used option, with strong versatility." This surfaces the "center dot" artefact that the deep dive didn't explicitly capture.

[^orca-issue-ironing-inset]: OrcaSlicer Issue #5812 — Ironing Inset. Community recommendation of 0.15-0.2 mm to prevent edge over-extrusion on perimeter-adjacent ironing paths. <https://github.com/SoftFever/OrcaSlicer/issues/5812>

[^orca-discussion-ironing-inset]: OrcaSlicer Discussion #2778 — Feature request context for the inset parameter. Describes the failure mode where ironing to the edge produces visible bulge or fuzz due to nozzle physical width. <https://github.com/SoftFever/OrcaSlicer/discussions/2778>

[^orca-wiki-ironing]: OrcaSlicer Wiki — Quality Settings → Ironing. Confirms the "Inset > 0: crisper edges and better-defined corners" rationale. <https://github.com/OrcaSlicer/OrcaSlicer/wiki/quality_settings_ironing>

[^bambu-matte-ironing]: Community calibration results for matte PLA variants (Bambu Lab forum and Facebook 3D printing groups, compiled 2026-04-11): Bambu Matte Black Speed 30 / Flow 25, Bambu Matte Grey Speed 30 / Flow 20, Overture Matte Speed 55 / Flow 15. Suggests matte PLA needs higher flow than standard PLA, but values are Bambu-hardware-specific and need local calibration before adoption. <https://forum.bambulab.com/t/ironing-percentage/172312>
