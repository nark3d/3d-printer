# Seams and Scarf Seams — comprehensive research dossier (v2 — 2026-04-11)

> **Goal**: complete reference document for everything we know about minimising and hiding seams in OrcaSlicer for PLA. This is the canonical *theoretical and empirical* deep-dive. The per-setting matrix lives in [`quality_PLA.md`](quality_PLA.md); the empirical case-study counterpart is [`seam_diagnosis_field_notes.md`](seam_diagnosis_field_notes.md).
>
> **v2 update (2026-04-11)**: expanded with ~5 agents of new research covering GitHub source code, every slicer's seam implementation, community real-user reports, YouTube creator settings, v2.3.2-specific bug analysis, commit-level version history, per-profile bundled comparisons, and post-processing options. **55+ references cited.**
>
> **Hardware context**: Phaetus Rapido 2 HF + Orbiter 2.5 + 0.4 mm nozzle on a CoreXY with input shaping (MZV X=61 Hz / Y=42 Hz). PLA, 0.2 mm layer height, quality goal. **OrcaSlicer 2.3.2** (verified from `/Applications/OrcaSlicer.app/Contents/Info.plist`).

---

## ⚡ TL;DR — what to do (if you don't want to read 2000+ lines)

**For this specific hardware + filament combo, on OrcaSlicer 2.3.2, for PLA quality-first scarf seams**:

| Setting | Recommended | Currently |
|---|---|---|
| `seam_slope_type` | `all` (Contour+Hole) | ✅ already set |
| `seam_slope_conditional` | `1` (true) | ✅ already set |
| `seam_slope_inner_walls` | `1` (true) | ✅ already set |
| `seam_slope_steps` | `20` | ✅ already set |
| `seam_slope_min_length` | `20` mm (default) — set **explicit** | ⚠️ currently inherits |
| `seam_slope_start_height` | **`10%`** (Bambu default) — set **explicit** | ⚠️ currently inherits 0 |
| `scarf_joint_speed` | **`75%`** (= 75 mm/s at our outer wall speed) | ⚠️ currently inherits 100 mm/s |
| `scarf_angle_threshold` | `155` (default) — set **explicit** | ⚠️ currently inherits |
| `scarf_overhang_threshold` | `40%` (default) — set **explicit** | ⚠️ currently inherits |
| `seam_slope_entire_loop` | `0` (default) — set **explicit** | ⚠️ currently inherits |
| `seam_gap` | **`0%`** (for scarf quality, per Noisyfox + begna112) | ⚠️ currently `5%` |
| `wipe_on_loops` | `0` — set **explicit** | ⚠️ currently inherits (fine, but not locked in) |
| `wipe_before_external_loop` | `0` — set **explicit** | ⚠️ currently inherits (fine, but not locked in) |
| `wall_sequence` | `inner-outer-inner wall` | ✅ already set |
| `wall_generator` | `arachne` (PR #9197 fixes the old Arachne+scarf bug in 2.3.1+) | ✅ already set |
| `staggered_inner_seams` | `1` | ✅ already set |
| `outer_wall_line_width` | `105%` (compromise; Adam L says 150% for best scarf) | ✅ set to 105% in earlier cleanup |
| `outer_wall_speed` | `100` mm/s (Adam L's range 75-100) | ✅ already set |
| `precise_outer_wall` | `1` | ✅ already set |
| `seam_position` | `aligned_back` | ✅ already set |

**The highest-impact fix** is changing `seam_gap` from `5%` to `0`. This is the single most common community finding for "my scarf seam is visible but I can't work out why" — and it's directly backed by Noisyfox (the implementation author) and begna112 (tested on Sunlu PLA+ X1C, found seam_gap=0 "mostly negates the inner scarf issue").

**Second-highest-impact fix** is adding explicit `scarf_joint_speed: 75%` and `seam_slope_start_height: 10%`. These two together bring scarf-start behaviour into the community-proven sweet spot.

See [Section 21 — Recommended settings for our hardware](#21-recommended-settings-for-our-hardware-final-revision) at the end of this document for the full rationale matrix.

---

## Table of contents

1. [Why seams are a problem](#1-why-seams-are-a-problem)
2. [The scarf seam concept](#2-the-scarf-seam-concept)
3. [OrcaSlicer source code — how scarf is actually implemented](#3-orcaslicer-source-code--how-scarf-is-actually-implemented)
4. [Version history: what changed between 2.1.1 → 2.3.2](#4-version-history-what-changed-between-211--232)
5. [Every OrcaSlicer scarf setting, in depth](#5-every-orcaslicer-scarf-setting-in-depth)
6. [Adam L's "Better Seams" guide — the core findings](#6-adam-ls-better-seams-guide--the-core-findings)
7. [The outer wall line width finding (contradicts Bambu wiki)](#7-the-outer-wall-line-width-finding-contradicts-bambu-wiki)
8. [Edge cases (sharp corners, overhangs, thin walls, random, spiral)](#8-edge-cases-sharp-corners-overhangs-thin-walls-random-spiral)
9. [Filament considerations (PLA, PETG, ASA/ABS, TPU)](#9-filament-considerations-pla-petg-asaabs-tpu)
10. [The wipe settings tension — scarf vs traditional](#10-the-wipe-settings-tension--scarf-vs-traditional)
11. [Real-user empirical data — the issue #7326 deep dive](#11-real-user-empirical-data--the-issue-7326-deep-dive)
12. [Real-user empirical data — discussion #4325 + misc forum reports](#12-real-user-empirical-data--discussion-4325--misc-forum-reports)
13. [PrusaSlicer scarf implementation — cross-slicer comparison](#13-prusaslicer-scarf-implementation--cross-slicer-comparison)
14. [SuperSlicer's alternative approach (seam_notch, wipe_inside)](#14-superslicers-alternative-approach-seam_notch-wipe_inside)
15. [Bambu Studio, Cura — what the other slicers do](#15-bambu-studio-cura--what-the-other-slicers-do)
16. [Community bundled profiles — per-profile comparison](#16-community-bundled-profiles--per-profile-comparison)
17. [YouTube creator settings and Hackaday coverage](#17-youtube-creator-settings-and-hackaday-coverage)
18. [Tuning workflow (start-to-finish)](#18-tuning-workflow-start-to-finish)
19. [Post-processing options (Unseam, G-code editing)](#19-post-processing-options-unseam-g-code-editing)
20. [Downgrade considerations — is 2.1.1 the answer?](#20-downgrade-considerations--is-211-the-answer)
21. [Recommended settings for our hardware (final revision)](#21-recommended-settings-for-our-hardware-final-revision)
22. [Normal seam tuning — the smallest gap without bulging](#22-normal-seam-tuning--the-smallest-gap-without-bulging)
23. [Open questions / things we couldn't pin down](#23-open-questions--things-we-couldnt-pin-down)
24. [References](#24-references)

---

## 1. Why seams are a problem

A seam is the visible artefact at the point where each layer's perimeter loop starts and ends. The physics are straightforward but unforgiving:

- The nozzle has to **start extruding** from a stop. Pressure has to build up in the melt chamber before any plastic actually comes out
- The nozzle then has to **stop extruding** at the end of the loop. Pressure has to bleed off, but the melt chamber is still under pressure when the motion stops
- The result: a small **under-extruded region** at the loop start (because pressure was building) immediately followed by **over-extrusion** at the loop end (because residual pressure squeezed out extra plastic)
- Both happen at the same physical location, layer after layer
- Stacked across hundreds of layers, this creates a **vertical scar** running the full height of the print

Pressure advance helps by predicting and compensating for this pressure dynamic, but it doesn't eliminate it. There is always *some* residual seam, and the better-tuned the printer, the harder it becomes to push the seam below visibility threshold [^klipper-pa].

The traditional approach is to **hide** the seam: put it where you can't see it. Aligned, Aligned Back, Back, Random — all of these are placement strategies. They don't make the seam smaller, they just put it somewhere less noticeable.

**Scarf seams take a different approach: instead of hiding the seam, eliminate the discontinuity that causes it.**

---

## 2. The scarf seam concept

The name comes from a [woodworking joint](https://en.wikipedia.org/wiki/Scarf_joint): two pieces of wood are joined by tapering each piece on a long diagonal, then overlapping the tapers. The result is a joint that's nearly invisible because there's no abrupt edge — the transition happens gradually over a long distance.

**MichaelJLew coined the term "scarf seam" for FDM** in PrusaSlicer issue #11621 [^prusa-issue-11621] — arguing it was the correct woodworking term over "sloped seam". This discussion seeded both the OrcaSlicer and PrusaSlicer implementations.

Applied to FDM:

1. Instead of starting the perimeter loop at full layer height with full extrusion, **start at zero height with zero extrusion**
2. **Ramp up** the Z height and the extrusion flow over a defined horizontal length (typically ~20 mm)
3. The loop runs around the perimeter as normal
4. When the loop returns to the start point, **continue past it** for a similar distance, this time **ramping down** to zero
5. The "extra material" that ramps down lays *on top of* the ramp-up section from the start, filling the missing volume

The end result: **no abrupt seam discontinuity** anywhere on the perimeter. The flow change is spread across ~20 mm of horizontal travel and 1 layer height of vertical travel. The wall is now an extruded *spiral* over the seam region rather than a vertical line.

The trade-off: the technique works *brilliantly* on **smooth curved surfaces** (where the ~20 mm ramp blends invisibly into the curve), and *less well* on **sharp corners and overhangs** (where the geometry forces a visible discontinuity anyway, or the ramp can sag).

---

## 3. OrcaSlicer source code — how scarf is actually implemented

**Implementation**: [PR #3839](https://github.com/SoftFever/OrcaSlicer/pull/3839) by Noisyfox, merged 2024-03-02 by SoftFever [^orca-pr-3839].

### 3.1 How the ramp is built (from `ExtrusionEntity.cpp/hpp`)

The scarf is implemented via a new `ExtrusionLoopSloped` class (+109/+52 lines added). The algorithm:

1. **Remember Z of previous layer** so the slope can ramp from a known starting point
2. **Travel to middle of layer Z** — enables sub-layer Z positioning during the ramp
3. **Support sloped extrusion** — the actual segmented Z + E ramping
4. **Disable seam gap for external perimeters** when scarf is enabled (the gap would create a visible discontinuity in the middle of the ramp)
5. **Slope length parameter** limits scarf application to perimeters longer than the specified value (otherwise the ramp would consume the entire perimeter, leaving no room for the actual loop)

**Ramp math** (simplified from the actual code):

```python
# For each segment 0..N-1 in the scarf slope:
height_fraction = start_height + (i / steps) * (1.0 - start_height)
e_fraction = (i / steps)  # linear from 0 to 1

# At segment i:
#   Z = previous_layer_Z + layer_height * height_fraction
#   E extrusion = normal_E * e_fraction
```

This produces a **linear double ramp**: height goes from `start_height × layer_height` up to `layer_height`, and extrusion goes from 0 up to full flow, both synchronised over `slope_min_length` horizontal mm divided into `slope_steps` segments.

### 3.2 The `seam_gap` interaction — critical

**`seam_gap` is applied at the END of the scarf slope via `clip_slope(seam_gap)`**, not at the start. When scarf is enabled, the code explicitly sets `clip_length = 0` for the outer perimeter so the normal seam-gap clip doesn't fire, and instead applies the gap via `clip_slope()` at the scarf tail.

**What this means in practice**: setting `seam_gap: 5%` on a scarf-enabled profile causes the slicer to clip 5% of the scarf slope's tail from the external perimeter. That clipping is visible as a slightly shortened end-of-slope, which on tightly-layered slopes can produce a visible micro-step or shelf in the seam region.

**Noisyfox's own comment in PR #3839**: "seam gap should not matter when this is enabled, as I've already disabled the seam gap for external perimeter when scarf is used. If seam gap does make a difference in your test, then that's a bug." [^orca-pr-3839]

**begna112's empirical confirmation** (issue #7326 comment, 2025-01-02, Sunlu PLA+ on Bambu X1C): "Disabling scarf seam gaps seems to mostly negate the inner scarf issue and lead to better quality." Before/after photos showed near-imperceptible seam with `seam_gap: 0`, visible bump with `seam_gap: 10%` [^orca-issue-7326].

### 3.3 Seam placement algorithm (`SeamPlacer.cpp`)

Seam placement uses a **Gaussian + sigmoid corner penalty function** [^orca-seam-placer-cpp]:

```cpp
float compute_angle_penalty(float ccw_angle) {
    return gauss(ccw_angle, 0.0f, 1.0f, 3.0f) + 1.0f / (2 + std::exp(-ccw_angle));
}
```

This means **concave corners** (negative ccw_angle, i.e., inside corners) get significantly lower penalty than convex corners. The `gauss` function with `falloff_speed=3.0` drops quickly away from 0. The combined curve gives sharper inside corners a much stronger preference as seam locations — which is why "sharpest corner" placement works: it puts the seam where the corner will visually absorb it.

**Commit history for `SeamPlacer.cpp`** (recent, relevant):
- `ecc2c91e` (2026-01-21): "Fix aligned back seam positioning for mirrored objects (#12028)"
- `2acd858b` (2025-07-29): "Introduce a new seam alignment option: Aligned back (#10255)" — this is the `aligned_back` option our profile uses
- `b665dfb3` (2024-05-23): "improve seam performance (#5436)"
- `dcb8d09a` (2024-03-27): "Add overhang threshold for scarf joint seam (#4725)"

### 3.4 Code-level defaults (PrintConfig.cpp) vs what our profile uses

From the current OrcaSlicer 2.3.2 `PrintConfig.cpp` [^orca-printconfig]:

| Key | Code default | Min | Max | Our profile |
|---|---|---|---|---|
| `seam_slope_type` | `none` | — | — | `all` ✓ |
| `seam_slope_conditional` | `false` | — | — | `1` ✓ |
| `scarf_angle_threshold` | 155° | 0 | 180 | (inherits) ⚠️ |
| `seam_slope_start_height` | 0 (absolute) | 0 | — | (inherits) ⚠️ |
| `seam_slope_min_length` | 20 mm | 0 | — | (inherits) ⚠️ |
| `seam_slope_steps` | 10 | 1 | — | `20` ✓ |
| `seam_slope_entire_loop` | `false` | — | — | (inherits) ⚠️ |
| `seam_slope_inner_walls` | `false` | — | — | `1` ✓ |
| `scarf_joint_speed` | 100% or 100 mm/s | — | — | (inherits) ⚠️ |
| `scarf_joint_flow_ratio` | 1.0 | 0 | 2 | (inherits — moved to dev mode v2.0.0+) |
| `scarf_overhang_threshold` | 40% | 0 | — | (inherits) ⚠️ |
| `seam_gap` | 10% | 0 | — | `5%` ⚠️ |
| `wipe_before_external_loop` | `false` | — | — | (inherits) ⚠️ |
| `wipe_on_loops` | `false` (default) | — | — | (inherits) ⚠️ |
| `seam_position` | `aligned` | — | — | `aligned_back` ✓ |

**The ⚠️ rows are settings our profile inherits silently from the parent chain.** Making them explicit in the profile JSON has two benefits: (a) the value survives future parent-profile changes, (b) it's visible in the profile file so anyone reading it knows what's actually active.

### 3.5 The BBL `fdm_process_common.json` reference

Bambu's own profile base overrides [^orca-bbl-fdm-common]:
- `seam_slope_min_length`: **10 mm** (vs code default 20)
- `seam_slope_start_height`: **10%** (vs code default 0)
- `seam_slope_type`: `none` (scarf off by default)
- `seam_position`: `aligned`
- `scarf_angle_threshold`: 155

**Critical for us**: the Custom/Klipper `fdm_process_common.json` in OrcaSlicer 2.3.2 does NOT set any scarf defaults — only `seam_position: aligned`. This means our Klipper-based profiles inherit the **code defaults**, not the **Bambu defaults**. In particular:
- Our inherited `seam_slope_start_height` is `0` (code default), not `10%` (Bambu default)
- Our inherited `seam_slope_min_length` is `20` (code default), not `10` (Bambu default)

This is a fork in the default path that matters for our tuning.

---

## 4. Version history: what changed between 2.1.1 → 2.3.2

This section is **critical for diagnosing a regression**. The user reports: "the last thing I printed had a pretty obvious scarf seam" and "I've had them basically perfect on my machine in the past". This is a classic symptom of either (a) a slicer version regression, or (b) a profile setting drifting out of the known-good state.

### 4.1 The cf6d9c7 regression — introduced in 2.2.0, REVERTED before 2.3.0

**This is a widely-known regression** that affected many users on 2.2.0. Tracking the full history:

**Commit cf6d9c7** [^orca-cf6d9c7], authored by Fritz Webering (@fritzw), merged into 2.2.0 with commit message: *"Avoid collisions when moving Z down — Avoid collisions with previous extrusions in the same layer when moving Z down in an XYZ move. This happens for example when starting a scarf joint after another perimeter was already printed. Fixes #7191."*

**What it intended**: the nozzle was occasionally crashing through already-printed walls when descending to the scarf start Z. The fix separated the Z movement from the XY travel, so the nozzle drops vertically first, then travels horizontally.

**What it broke (Issue #7326** [^orca-issue-7326] **— 80 comments)**: the nozzle now hovers momentarily above the scarf start, then plunges down and squashes the tiny remnant of molten filament on the nozzle tip, expanding it horizontally into a blob. With normal (non-scarf) Z starts the cross-section is large enough to absorb this; with a scarf start at ~10% of layer height, there is almost no clearance and the blob is very visible.

**@shyblower's mechanism analysis** (issue #7326, 2025-01-01 and 2025-01-15):
> "during retraction a very small amount of already printed material from the end of the previous line gets also sucked back into the nozzle and thus is pushed out again at the beginning of the next line. This amount is so small that it does not matter too much when the next line is drawn with the same vertical thickness as the previous one (the configured layer height) but since the nozzle starts the next line very closely positioned over the previous layer when the scarf seam is applied, this unintentionally carried and unretracted extra material produces the blob since it has to expand far more."

**What happened at 2.1.1**: the nozzle travelled diagonally through the inner perimeter (a physical collision with inner walls) on the way to the scarf start, which *wiped the nozzle tip clean*. This was an **accidental wipe**. The "fix" in 2.2.0 prevented the collision but also removed the wipe, revealing the underlying blob.

**Revert status**: **PR #8478** merged into main on **2025-02-21** [^orca-pr-8478], reverting cf6d9c7. The revert is **present in v2.3.0 and therefore in v2.3.2**. **This bug is NOT present in our current 2.3.2.** Confirmed by user pm0u (2025-05-01) and fritzw (2025-05-22): "I printed a test on 2.1.1 and on 2.3.0 and they look identical."

**Therefore**: cf6d9c7 is NOT the cause of our recent regression. The blob bug has been fixed in our version. Something else is at play.

### 4.2 Release-by-release scarf changes (complete history)

| Version | Date | Seam-related change | Effect on our setup |
|---|---|---|---|
| **2.0.0** | 2024-07 | `scarf_joint_flow_ratio` moved to **developer mode only** — testing by @psiberfunk found "its utility remains unclear" [^orca-2-0-release] | No user-visible change unless dev mode enabled |
| 2.1.1 | 2024-09 | Gold-standard scarf reference version (no known regressions) | — |
| **2.2.0** | 2024-10 | **cf6d9c7** introduced the blob bug (separated Z+XY travel before scarf start) | Affected 2.2.0 through the revert |
| **2.2.x → 2.3.0** | 2025-02-21 | **PR #8478** reverted cf6d9c7 | **Blob bug gone from 2.3.0+** |
| 2.3.0 | 2025-03 | No other seam-specific fixes | — |
| **2.3.1-alpha** | 2025-04 | **PR #9197** [^orca-pr-9197]: "Avoid unnecessary travel in scarf seam" — fixes Issue #9139 (Arachne + scarf = Z-hop + blob) | **Important**: we use Arachne + scarf, so this fix benefits us |
| 2.3.1 | 2025-10 | PR #10803: "Enhance GCode handling for Z-axis movements" | Small motion-planner improvement |
| **2.3.2-beta** | 2026-02 | **PR #10722** [^orca-pr-10722]: "Reduce artifacts from short travel moves before external perimeters — short travel moves now use outer wall acceleration instead of travel acceleration" | Helps with seam-adjacent vibration artefacts |
| 2.3.2-beta | 2026-02 | PR #12028: "Fix aligned back seam positioning for mirrored objects" | Matters if we print mirrored objects |
| 2.3.2-rc | 2026-03 | PR #11923: "Dual seam fuzzy painted" fix | Fixes fuzzy skin + seam painting edge case |
| 2.3.2-rc2 | 2026-03 | PR #12632: Apply fuzzy-skin double-seam fix to classic wall generator | — |
| **2.3.2 final** | 2026-03 | PR #12364: "Fix hidden line type markers (wipe, seam, retract)" — preview UI only | Visual fix in preview, no gcode change |

### 4.3 Open issues on v2.3.2 that affect scarf seams

**Issue #13068** [^orca-issue-13068] (2026-04-01, open): "When scarf is enabled, seam placement shifts location by one scarf length from expected position." — v2.3.2 specifically. With scarf ON, the visible seam mark shifts by exactly one scarf length from where the slicer's preview shows it. With `aligned_back` seam position (which we use), this could mean the visible artifact lands a scarf-length away from the rear face.

**Workaround**: either (a) disable "Reverse on Even" for external perimeters (avoids the double-track variant), or (b) manually paint the seam one scarf-length before where you want the visible landing.

**Issue #12049** [^orca-issue-12049]: "Scarf on protruding parts degrades surface quality" — causes stripe artefacts where geometry transitions. Only affects models with secondary features (bosses, arms, gussets) protruding from a main body. **Simple cylinders / rectangles / prints without intersecting features are not affected.**

**Issue #12270** [^orca-issue-12270]: Conditional overhang threshold not respected. Scarf still applies to overhang perimeters regardless of `scarf_overhang_threshold` setting. Workaround: disable Conditional Scarf and manually exclude overhangs via seam painting.

**Issue #7892** [^orca-issue-7892]: Adaptive Pressure Advance is **incompatible with scarf seams** — closed NOT_PLANNED. "Scarf seam sets varying flow and layer height almost continuously and adapting the PA to match this would cause extruder stutter." **Our profile has `adaptive_pressure_advance: 0`** so this doesn't affect us, but it's a landmine for anyone who turns Adaptive PA on while using scarf.

---

## 5. Every OrcaSlicer scarf setting, in depth

### 5.1 `seam_slope_type` — Scarf joint seam

| Option | Behaviour |
|---|---|
| **None** | No scarf, traditional seams everywhere |
| **Contour** | Scarf only on the *outermost* perimeter |
| **Contour and Hole** *(recommended for quality)* | Scarf on outer perimeter *and* on perimeters around holes |

**Recommendation**: `Contour and Hole` for quality goal. Holes are visible too — scarf-ing them gives consistent visible quality everywhere.

**Trade-off**: marginally more print time vs Contour-only because the slicer applies the technique to more loops.

**Our value**: `all` (which maps to Contour and Hole). ✅ already correct.

### 5.2 `seam_slope_conditional` — Conditional scarf joint

| Value | Behaviour |
|---|---|
| `false` | Apply scarf to *every* perimeter loop, regardless of geometry |
| `true` *(recommended)* | Apply scarf only on *curved* perimeters (above the angle threshold). Sharp corners use traditional seams |

**Why conditional matters**: scarf seams blend beautifully into curves, but they don't help on sharp corners — at a 90° corner the scarf ramp can't blend because the geometry has a hard discontinuity anyway. Worse, ramping over a sharp corner can make the corner look *weird* because the ramp starts on one face and ends on the perpendicular face.

**bdwilson's late-breaking finding** (issue #7326, 2025-12-30, the final comment before auto-close):
> "This works great on both my Snapmaker U1 and my Bambu A1 printers using latest Orca. The biggest change: **Conditional scarf seams were enabled** (`seam_slope_conditional = 1`)"
>
> Old profile (2.1.1 style): `seam_slope_conditional = 0` (scarf always tries to ramp regardless of geometry)
> New working profile (2.3): `seam_slope_conditional = 1` (scarf only activates when perimeter is long enough and the corner angles allow)

**This is a key finding**: many profiles that were originally created on 2.1.1 had `seam_slope_conditional = 0`. Importing or copying those profiles to 2.3.x without updating them leaves scarf firing on short perimeters where it produces a bad blob. **Our profile already has this set to `1`** — we're on the right side of this fix.

**Our value**: `1` (true). ✅ already correct.

### 5.3 `scarf_angle_threshold` — Conditional angle threshold

The angle (measured at the perimeter corner) below which a corner is considered "sharp" and scarf is skipped.

| Value | Behaviour |
|---|---|
| **155° (default)** | Anything more obtuse than 155° is treated as a curve and gets scarf'd. Sharp 90° and 120° corners stay traditional |
| Higher (e.g. 170°) | Only the smoothest curves get scarf. Most corners stay traditional |
| Lower (e.g. 115°) | Most corners get scarf, including moderate angles. Adam L uses this for testing |

**Adam L's specific setup**: he uses conditional scarf at threshold 115° for testing (which forces scarf on more than the default 155°), but explicitly recommends 155° for production use.

**Recommendation**: leave at 155° for production. The default is well-tuned.

**Our value**: inherits code default 155 ⚠️. **Recommend making explicit** to lock it in.

### 5.4 `scarf_overhang_threshold` — Conditional overhang threshold

The overhang percentage above which scarf is skipped (because scarf seams sag on overhangs).

| Value | Behaviour |
|---|---|
| **40 % (default)** | Skip scarf where the perimeter has more than 40 % overhang above |
| Higher | Allow scarf on more overhanging features. Risk of sag |
| Lower (e.g. 25 %) | Stricter — skip scarf on even small overhangs |

**Recommendation**: leave at 40 %. Tuning this is rarely productive.

**Gotcha**: **Issue #12270** reports this setting is not always respected — scarf fires on overhanging regions regardless. If you see scarf sagging on overhangs, the workaround is to disable Conditional Scarf entirely and rely on seam painting.

**Our value**: inherits 40%. **Recommend making explicit** for profile clarity.

### 5.5 `scarf_joint_speed` — Scarf joint speed

The speed at which the scarf ramp itself is printed.

| Value | Behaviour |
|---|---|
| **100 % (default)** | Scarf ramp prints at the same speed as the perimeter feature it's part of |
| Lower (e.g. 80 %) | Scarf ramp prints slower than the rest of the perimeter |
| Absolute value (e.g. 75 mm/s) | Override to a specific speed regardless of perimeter speed |

**Adam L's findings — this is critical** [^adam-l-guide]:

- "Optimal speeds are generally in the range **50-100 mm/sec** for the scarf speed"
- His settings use 75 or 100 mm/sec
- "Critical for avoiding wall distortions"
- On overhangs: 75 mm/sec produced OK results, 100 mm/sec produced *better* results, but **150 mm/sec was far worse**
- "Sweet spot" varies by filament/temperature/hotend — needs testing

**The OrcaSlicer wiki agrees**: "Recommended to print scarf joints at a slow speed (less than 100 mm/s)" [^orca-wiki-seam].

**Noisyfox's own photo evidence from PR #3839**: "Reducing the outer wall speed greatly improves the scarf quality" — with high-speed (outer wall first) = visible artifact, low-speed = nearly invisible [^orca-pr-3839].

**Anycubic Kobra S1 bundled profile** uses `scarf_joint_speed: 35` (absolute mm/s) — very slow, prioritising quality over time.

**Community "Fine" profile**: our own `0.12mm Fine @MyKlipper - Customised` profile has `scarf_joint_speed: 50%` explicitly set, reflecting a deliberate "slower for quality" choice.

**Recommendation for our hardware**: at `outer_wall_speed: 100 mm/s`, the default 100% scarf multiplier puts us *exactly at* Adam L's upper-limit 100 mm/s. **There's no safety margin.** Recommend setting to `75%` (= 75 mm/s) to move to the middle of Adam L's sweet spot.

**Our value**: inherits code default 100%. **Recommend 75%** explicit.

### 5.6 `seam_slope_start_height` — Scarf joint height

Vertical offset where the scarf ramp starts. Set in mm or as a percentage of layer height.

| Value | Behaviour |
|---|---|
| **0 (code default)** | Ramp starts at the current layer Z and rises to full layer height over the scarf length |
| **10% (Bambu default)** | Ramp starts at 10% of layer height above the previous layer |
| 50% | Ramp starts at 50% of layer height (significantly less clearance before the ramp) |
| Higher | Ramp starts above current layer height (creating a steeper transition) |

**Why this matters**: at `start_height = 0`, the first scarf segment has essentially zero clearance from the previous layer. At 0.2 mm layer height with 10 steps, segment 1 has Z = 0.02 mm — the nozzle is physically touching the previous layer. Any tiny filament droplet on the nozzle tip at that moment gets squashed into the previous layer, creating a visible blob.

At `start_height = 10%`, segment 1 has Z = 0.02 + 10%×0.2 = 0.04 mm — double the clearance. The droplet has more room to form cleanly before the nozzle contacts the previous layer surface.

**Our 2024 known-good profile** used `seam_slope_start_height: 50%` specifically to fix a "knotty mess at scarf entry" artifact (see [`seam_diagnosis_field_notes.md`](seam_diagnosis_field_notes.md) Case 4).

**vgdh's recommendation** (discussion #4325) [^orca-discussion-4325]: "start height 0.1 and seam gap = 0" — specifically for first-layer adhesion issues on scarf.

**Recommendation**: **set to `10%`** as the default starting value. This matches Bambu's default, is far more proven than code-default 0, and gives the scarf entry blob mechanism less material to work with.

**Our value**: inherits code default 0 ⚠️. **Recommend 10%** explicit.

### 5.7 `seam_slope_entire_loop` — Scarf around entire wall

| Value | Behaviour |
|---|---|
| **false (default)** | Scarf is only at the start and end of the perimeter loop (the seam region). Most of the loop is normal printing |
| `true` | Scarf wraps around the entire perimeter loop. The nozzle does *two passes* — one as the spiral ramps up, one as it ramps down |

**Adam L's verdict**: "Has not significantly improved results for me, but some people claim it works better"

**Trade-off**: huge print time cost (you're printing every perimeter loop twice), marginal-at-best quality gain.

**Gotcha**: Issue #5465 reports that "entire loop" mode causes flow rate to drop below the volumetric limit, causing rough outer layers. DaveMeade5: "The scarf joint pushes the volumetric flow almost down to zero."

**Recommendation**: leave off. Only enable if you've tried everything else and need the absolute best seam.

**Our value**: inherits false. **Recommend making explicit** for profile clarity.

### 5.8 `seam_slope_min_length` — Scarf joint length

The horizontal distance over which the scarf ramp transitions from zero to full extrusion. Loops shorter than 2× this value can't be scarf'd at all (no room for the ramp).

| Value | Behaviour |
|---|---|
| 0 | Disables scarf joint entirely |
| **20 mm (code default)** | Standard ramp length, good for most prints |
| **10 mm (Bambu default)** | Tighter ramp, catches smaller features |
| 12 mm (Creality Sermoon scarf profile) | Between the two |
| Longer (e.g. 30 mm) | Smoother ramp, but consumes more of the perimeter. Bad on small features |

**Adam L's specific finding**: "Making the scarf length shorter than the default 20 starts to become a particular quality problem. I recommend leaving it at default of 20."

**BUT**: the Bambu bundled profiles use **10 mm**, not 20 mm. And Anycubic Kobra S1 / Phrozen Arco use 10 mm. This is a significant difference: Adam L tested 20 mm on an X1C and recommends it, but the X1C's own bundled profile uses 10 mm. The most likely explanation is that 10 mm is better for **small features** (scarf fires on smaller perimeters) at the cost of slightly more visible scarf on large features.

**Recommendation**: stay at 20 mm for our quality goal. Small features are less common in our typical prints, and the cleaner ramp on large features matters more. If we start hitting "scarf not applied to small curved features", try 10-12 mm.

**Hidden caveat**: on very small loops (e.g. a 30 mm circumference cylinder), 20 mm of scarf consumes most of the loop. The slicer falls back to traditional seams when the loop is too short. This is *one of the main reasons* you might see scarf working on a large model and not on small features in the same print.

**Our value**: inherits 20 mm. **Recommend making explicit**.

### 5.9 `seam_slope_steps` — Scarf joint steps

The number of discrete segments used to build the scarf ramp.

| Value | Behaviour |
|---|---|
| 5 | Coarse ramp, possibly visible faceting |
| **10 (default)** | Sufficient for most prints — Adam L confirms |
| 20 | Smoother, more gcode lines, marginal quality gain |
| 30+ | Diminishing returns |

**Adam L's verdict**: "Changing the scarf steps doesn't seem to matter much. The default 10 is fine."

**Our value**: `20`. We're being slightly aggressive with smoothness. Adam L would say this is unnecessary but harmless. ✅ keep as-is.

**Note**: Creality Sermoon scarf profiles use `seam_slope_steps: 6`. This is aggressive the other direction — fewer steps = coarser ramp = slightly more visible scarf but simpler gcode.

### 5.10 `scarf_joint_flow_ratio` — Scarf joint flow ratio

Multiplier on extrusion flow during the scarf ramp. Idea: reduce flow during the ramp to compensate for the over-extrusion that traditional seams cause.

| Value | Behaviour |
|---|---|
| **100 % (default)** | Standard flow during scarf, no adjustment |
| 95% | Creality Sermoon uses this — slightly reduced to compensate for cumulative over-extrusion |
| 90 % | Slightly reduced flow during the ramp |
| 80 % | More aggressive reduction |

**Adam L's finding**: "Changing the scarf flow ratio doesn't seem to help. I tested down to 90 % flow without improvement."

**Adam L specifically collaborated with SoftFever to add this feature**, hoped for improvement, found none in his testing. The setting was **moved to developer mode only in OrcaSlicer 2.0.0** because "its utility remains unclear based on testing results" [^orca-2-0-release].

**Caveat**: a different community user (FuryriderX in PR #3839 testing) reported success on PETG at 0.8 (80 %) flow ratio after 40+ attempts. So **for PETG specifically**, this *might* be tunable. For PLA on a well-calibrated Rapido 2 HF, leave at 100 %.

**Our value**: inherits 100% (dev mode only anyway). ✅ leave alone.

### 5.11 `seam_slope_inner_walls` — Scarf joint for inner walls

| Value | Behaviour |
|---|---|
| **false (default)** | Scarf only on outer / hole perimeters |
| `true` | Scarf also on inner perimeters |

**Adam L's verdict**: "Often (but not always!) improves results, but will slow down the print overall"

**Why it sometimes helps**: if your inner wall has visible ghosting or print-through artefacts, scarf-ing the inner walls reduces the inner-wall seam, which reduces the visual artefact on the outer surface caused by the inner-wall seam being right behind the outer wall.

**Why it sometimes hurts**: on overhangs, "the hot nozzle sits there while de-retraction happens, leaving visible inner smoosh". The inner scarf ramp can mark the inside of the wall in a way that's then visible on the outside on overhangs.

**Critical empirical finding from our own printing** (see [`seam_diagnosis_field_notes.md`](seam_diagnosis_field_notes.md) Case 1): enabling scarf inner walls **fixed a 1-2 second dead-stop pause at the seam location on large ASA models**. The inner wall had a conventional hard-stop seam that created a steep pressure cliff requiring the toolhead to decelerate. Enabling scarf on inner walls eliminated the cliff. **On large prints, scarf inner walls is a functional necessity, not just a quality improvement.**

**Our value**: `1` (true). ✅ already correct — and now justified by both Adam L's guide AND our own field experience.

---

## 6. Adam L's "Better Seams" guide — the core findings

The single most thorough community resource on scarf seams is [Better Seams: An Orca Slicer Guide to Using Scarf Seams](https://www.printables.com/model/783313-better-seams-an-orca-slicer-guide-to-using-scarf-s) by **Adam L / @psiberfunk** on Printables [^adam-l-guide]. The guide is the result of:

- **60+ experiments** on simple geometry (cylinders, double hexagons)
- **80+ experiments** on a custom curved-pot model designed specifically to stress-test overhang behaviour
- All testing done on **Bambu X1C with 0.4 mm nozzle**
- Tuned against **Prusament Simply Green PLA** (specific filament reference matters)
- Collaboration with **SoftFever** (OrcaSlicer maintainer) and **Michael at Teaching Tech**
- Published alongside a YouTube video by Teaching Tech [^teaching-tech-video]

**The guide is the de facto reference** for scarf seam tuning. Below is everything actionable from it.

### 6.1 Adam L's specific recommendations

| Setting | Recommended value | Reasoning |
|---|---|---|
| **Outer wall speed** | **75-100 mm/s** | "Optimal range for scarf speed". Critical for avoiding wall distortions |
| **Outer wall line width** | **0.6 mm (150 % of nozzle)** for 0.4 mm nozzle | **Wider walls decrease extrusion rate error**. This is one of the most surprising and important findings in the guide |
| **Layer height** | **0.2 mm** | All testing was done at 0.2 mm. Higher layers may help but can hurt — needs per-print testing |
| **Wall order** | **Inner-outer-inner** | "Strongly recommended". Outer/inner provides worse results, as does inner/outer |
| **Extrusion rate smoothing (`max_volumetric_extrusion_rate_slope`)** | **300 mm³/s²** on Bambu X1C | "Non-X1/P1 printers may need other settings" |
| **Scarf length** | **20 mm (default)** | Shorter "starts to become a particular quality problem" |
| **Scarf steps** | **10 (default) is fine; he uses 20** | Doesn't matter much |
| **Conditional scarf joint** | **ON** (strongly recommended) | "Apply scarf seam only when a round surface (vs sharp corner) is encountered" |
| **Scarf joint type** | **Contour and Hole** | "Contour mode will give you scarf seams on the outermost wall only; Contour+Hole will include inside hole walls" |
| **Scarf flow ratio** | **100 % (default)** | "Doesn't seem to help" — tested down to 90 % without improvement |
| **Scarf for inner walls** | **`true` for quality, but slower** | "Often improves results" but slows the print |
| **Scarf around entire wall** | **OFF** | "Has not significantly improved results for me" |
| **Scarf start height** | **10-50% of layer height** | Bambu default 10%; Adam L testing found 50% helps for blob-at-entry issues |

### 6.2 Adam L's "what to AVOID" list (the most important practical section)

Settings that **make scarf seams worse**:

1. **`wipe_on_loops: true`** — "Using Wipe on loops makes things bad"
2. **`wipe_before_external_loop: true`** — "Wipe on external loops make things bad"
3. **`retract_on_layer_change: false` AND `wipe_while_retracting: false` together** — "No good"

**This is critical** because the OrcaSlicer wiki *recommends* `wipe_on_loops: true` for general use. But the wiki recommendation is for *traditional* seams. **For scarf seams, Adam L's testing says to turn it OFF.** These two pieces of guidance are in direct conflict — Adam L's empirical testing wins for scarf seam users.

**Our current state**: we inherit `wipe_on_loops: false` (matches code default) and `wipe_before_external_loop: false` (matches code default). **We're correct**, but the values aren't explicit in the profile JSON. Recommend making them explicit.

**Our printer profile** now has `retract_when_changing_layer: 1` and `wipe: 1` (both enabled, as of today's cleanup pass) — this satisfies Adam L's rule #3.

### 6.3 Adam L's test methodology

The "TestSuite_OrcaSlicerOnly_V6.3MF" file from the Printables guide includes:

- Three test objects per setting set: cylinder, double-hexagon, and small curved pot
- **Two optimised setting sets** + one traditional-seam reference for comparison
- All tuned against Prusament Simply Green PLA

**Workflow Adam L recommends**:

1. Use OrcaSlicer 2.0 beta or newer (we're on 2.3.2)
2. Set up your filament profile (temperature, flow, pressure advance) FIRST — scarf seam tuning depends on these being right
3. Print the test suite on your printer
4. Look at the results and pick the best settings
5. Tweak from there based on what you see

---

## 7. The outer wall line width finding (contradicts Bambu wiki)

Adam L recommends **0.6 mm outer wall on a 0.4 mm nozzle** — that's **150 % of nozzle diameter**. The reasoning: "Wider walls decrease extrusion rate error".

This is **directly opposed to general OrcaSlicer wiki guidance**, which recommends 100 % of nozzle diameter for outer wall (for "best dimensional accuracy") [^orca-wiki-line-width].

**The trade-off is real:**

| Optimisation goal | Recommended outer wall width |
|---|---|
| **Best dimensional accuracy** (holes match design size) | 100 % (0.4 mm) |
| **Best general surface finish** (Bambu Lab wiki) | 105-120 % (0.42-0.48 mm) |
| **Best scarf seam quality** (Adam L specifically) | **150 % (0.6 mm)** |

You can't have all three. The choice depends on what you're prioritising.

**For our quality-first PLA goal with scarf seams enabled**, Adam L's 150 % is the correct recommendation. The dimensional accuracy hit (~0.1 mm on hole sizes) is acceptable; the scarf seam quality gain is significant.

**Our current value**: `105%` (0.42 mm) — set during the earlier cleanup pass based on the general Bambu wiki recommendation.

**Decision**: **keep 105%** as a compromise. Going to 150% would be more scarf-optimal but sacrifices too much dimensional accuracy for general-purpose prints. If we hit a specific scarf-critical print, we can create a separate profile variant with 150% outer wall width.

This choice is documented in [`quality_PLA.md`](quality_PLA.md) and [`seam_diagnosis_field_notes.md`](seam_diagnosis_field_notes.md) as a deliberate trade-off.

---

## 8. Edge cases (sharp corners, overhangs, thin walls, random, spiral)

### 8.1 Sharp corners

Scarf seams cannot meaningfully blend at a sharp corner because the geometry itself has a hard angular discontinuity. The conditional scarf logic (`seam_slope_conditional: true`) is the right answer — it skips scarf on corners and uses traditional seams there. Trying to scarf a sharp corner often produces visible artefacts on adjacent faces.

### 8.2 Overhangs (the hardest case)

This is where scarf seams *can* fail. Adam L spent ~80 experiments on overhang testing.

**The problem**: scarf ramps print "in the air" without support if the perimeter is at an overhang. The hot nozzle's drag against unsupported plastic causes sag. The sag is visible.

**Adam L's findings on overhangs**:

1. **Speed sweet spot is narrow**: 75 mm/s OK, 100 mm/s better, 150 mm/s much worse
2. **Curved overhangs are particularly challenging**
3. **Single or double walls with no infill perform best** for thin curved features (more material in the wall stack creates more sag risk)
4. **Conditional overhang threshold (40 % default)** does most of the work — scarf is automatically skipped on the worst overhangs
5. **Inner scarf joints can hurt** on overhangs because of the de-retract smoosh — sometimes turning off `seam_slope_inner_walls` improves overhang results

**Most important advice**: **paint the seam away from the overhang in the slicer prepare view**. OrcaSlicer supports per-model seam painting — you can manually move the seam to a non-overhanging part of the perimeter. Adam L explicitly recommends this when the geometry allows.

### 8.3 Thin walls / no infill

Adam L found: "Single or double walls with no infill performed the very best" for the curved-pot test model. This is counterintuitive — you'd think more walls = more material = better strength = better seam. In practice, thin-wall models with no infill have less mass to fight against scarf-induced flexing during cooling.

If you have a part where scarf seams look bad, consider whether reducing the wall count or infill could improve the result. Quality-first ≠ thickest-walls-possible.

### 8.4 Random seam position

**Random seam placement is incompatible with scarf seams.** Noisyfox specifically called this out in PR #3839: random + scarf produces "blobs and strings". This is because the random position changes the seam location every layer, so the scarf ramp lands on a different point each time, creating visible scarf-on-scarf transitions all over the print.

**If you're using scarf seams**, use Aligned, Aligned Back, Back, or Nearest. **Never Random.**

### 8.5 Spiral mode

Spiral vase mode is fundamentally incompatible with scarf seams because spiral mode *is* a continuous Z spiral with no per-layer seams. Enabling both at once is meaningless. OrcaSlicer should silently disable scarf when spiral mode is on.

### 8.6 Arc fitting interaction

`Tikonderoga` in discussion #4325 notes: "Arc fitting is disabled when scarf seams are enabled." If you're a Klipper user with `[gcode_arcs]` enabled and you've turned on scarf seams, you're not actually getting arc-fitting gcode for the scarf region — it's converted to line segments for the slope. This is the correct behaviour (you can't fit an arc through a spiralling slope), but worth knowing.

---

## 9. Filament considerations (PLA, PETG, ASA/ABS, TPU)

### 9.1 PLA — well-supported

Adam L's testing was specifically on Prusament Simply Green PLA. PLA is the most cooperative material for scarf seams because:

- Predictable cooling behaviour
- Good layer adhesion at typical speeds
- Forgiving of minor flow variations

His settings should transfer reliably to most PLA brands. Re-tune `pressure_advance` and `flow_ratio` per filament, but the scarf seam process settings can stay constant.

**Matte PLA caveat**: our Sunlu PLA Matte has higher viscosity than standard PLA due to filler additives. This means:
- PA may need to be slightly higher (our calibrated value is 0.045, vs ~0.035 for glossy PLA on the same hardware — see [`sunlu_pla_matte.md`](../filament/sunlu_pla_matte.md))
- Scarf ramp at 0.02 mm effective layer height may not adhere as well at the start
- **No specific matte PLA scarf data in the community** — this is an area where we're flying somewhat blind

### 9.2 PETG — finicky

PETG was specifically called out by Noisyfox in PR #3839 as "showing inconsistent results requiring per-filament parameter tuning". **FuryriderX's testing on PETG** (PR #3839 comments):
> "I had to set seam gap to 0 and disable wipe on loops and wipe on external loops."

With those settings, near-invisible seam. Otherwise visible artifact.

**Community PETG-specific recommendations**:

- Lower scarf flow ratio to 0.8 if you see over-extrusion at the ramp (enable dev mode to access this)
- Reduce scarf speed to the lower end of the range (50-75 mm/s)
- Pressure advance must be very well tuned
- Try negative extra length on restart: -0.1 to -0.2 mm (per shyblower's testing on PCTG)

PETG is harder because:
- Higher viscosity = slower pressure dynamics = pressure advance more critical
- Stringy by nature = wipe interactions with scarf are more visible
- Cools slower than PLA = scarf ramp can sag if speed is too high

### 9.3 ABS / ASA — data exists from Voron community

**JendaDH's Voron 2.4 + Phaetus Dragon UHF + Prusament ASA** (Voron forum) [^voron-jendadh]:
- Extrusion width: 0.4 mm
- Extrusion multiplier: 93-95%
- Pressure advance range: 0.038-0.052 (wide tuning range)
- Print temperature: 265°C
- Result: "resolved the gap in perimeter by tuning PA and smooth timing"

**stevereno30's Voron 2.4 + Dragon HF + ABS** (Voron forum) [^voron-stevereno]:
- PA 0.020: "Corner bulging on trailing edges"
- PA 0.026: "significant gapping appears"
- PA 0.032: Bulging reduced but gaps starting
- PA 0.034: Best for bulging but "gaps are pretty awful"
- PA 0.040: Eliminated bulging but "dramatic corner perimeter gaps"
- **Conclusion**: high-flow hotends on ABS may show a PA sweet-spot problem where no single value fixes both bulge and gap. Solution: **adjust overlap/seam_gap settings instead of fighting PA further**. Set extrusion multiplier to 1.0, perimeter overlap in the 70s range, solid infill overlap around 30s.

**pm0u's 2.3.2 working ABS profile** (issue #7326, 2025-05-23):
- Max flow 24 mm³/s
- Outer walls 200 mm/s (high)
- Scarf speed 40% (= 80 mm/s)
- Z hop on retraction 0.4 mm
- Retract on layer change ON
- Wall order inner/outer
- 4 walls, 0.2 mm layers
- Result: "a pretty good result" — confirmed working

### 9.4 TPU — probably won't work

TPU's compliance and pressure-advance characteristics make scarf seams unlikely to work well. Stick to traditional seams for TPU.

---

## 10. The wipe settings tension — scarf vs traditional

### 10.1 The problem

There's a real, unresolvable tension between scarf seams and traditional seams:

- **`wipe_on_loops` improves traditional seams** — it tucks the loop end inward to hide the seam point
- **`wipe_on_loops` hurts scarf seams** — it interferes with the scarf ramp and creates artefacts

If you have a print where *some* features use scarf and *some* use traditional (e.g. mixed curved and sharp features with `seam_slope_conditional` doing the per-feature decision), you can't optimise for both at once. You have to pick.

### 10.2 How to decide

| If your print is... | Set `wipe_on_loops` to |
|---|---|
| Mostly curved surfaces (scarf will be active most of the time) | **`false`** |
| Mostly sharp corners (scarf rarely fires, traditional seams are dominant) | **`true`** |
| Mixed | **`false`** if visible quality matters more than functional strength on the corners |

**shyblower's confirmation** (issue #7326, 2025-02-27): "I tested `wipe_before_external_loop: true` with scarf seams. It does not help — same result." This settles the question for that specific setting.

### 10.3 The 2024 backup profile worked around this

Adam's known-good 2024 profile **avoided the tension entirely** by running scarf on *everything* (`seam_slope_type: all`, no conditional disabling). It never printed a traditional seam at all, so the wipe_on_loops conflict never arose. **This is a defensible profile design choice** — accept that scarf will fire on every perimeter, including sharp corners where it's slightly suboptimal, in exchange for never having to deal with the wipe configuration conflict.

### 10.4 Our current state

Our profile has:
- `seam_slope_type: all`
- `seam_slope_conditional: 1`
- `wipe_on_loops`: inherits `false` (correct, but implicit)
- `wipe_before_external_loop`: inherits `false` (correct, but implicit)

**Recommendation**: make the wipe settings **explicit false** in the profile so they're locked in against any future parent profile changes. This is a zero-risk cleanup.

---

## 11. Real-user empirical data — the issue #7326 deep dive

Issue #7326 [^orca-issue-7326] is the most complete public record of scarf seam debugging. It has 80 comments spanning 2024-11-01 to 2025-12-30. Here are the key data points.

### 11.1 shyblower (Thomas Scheiblauer) — the mechanism analyst

**shyblower** is the person who traced the blob bug to commit cf6d9c7 and helped orchestrate the revert. Their testing spanned multiple versions.

**Testing on 2.2.0 (affected version)**:
- Old profile with `wipe_on_loops: true`: "made it WORSE" on 2.2.0 (opposite of 2.1.1)
- Enabled "Retract on layer change" + "Wipe while retracting" + wipe distance ~0.8-1 mm: helped temporarily but later stopped working
- **Negative "Extra length on restart" of -0.05 to -0.10 mm for PLA**: effective
- Tested `wipe_before_external_loop: true` specifically: "Does not help — same result"

**Testing on 2.3.0 (after PR #8478 revert)**:
- Compared 2.1.1 vs 2.3.0 side by side: "they look identical" — confirming the blob bug is gone

**shyblower's diagnosis quote**:
> "Seems to be caused by the priming (unretract) before starting the scarf slope... during retraction a very small amount of already printed material from the end of the previous line gets also sucked back into the nozzle and thus is pushed out again at the beginning of the next line... produces the blob since it has to expand far more in horizontal direction due to the very small gap between nozzle and the layer below."

### 11.2 begna112 — the Sunlu PLA+ X1C tester

**begna112**, 2025-01-02, **Sunlu PLA+ on Bambu X1C HF with 0.4 nozzle**, pressure advance 0.2 (Bambu unit), line width 0.42 mm:

**Tested settings**:
- Negative "Extra length on restart" = -0.21 (= half line width / 2)
- `seam_gap: 0` (from default 10%)
- Wall order: inner/outer/inner
- Result: **"nearly perfect results on outer walls"**

**begna112's key finding**:
> "Disabling scarf seam gaps seems to mostly negate the inner scarf issue and lead to better quality."

**This is the single most directly relevant finding for us.** Sunlu PLA on a high-flow hotend with direct drive — basically our setup (except Bambu X1C vs Klipper). And the fix was `seam_gap: 0` + negative extra length on restart.

**Before/after photos** showed near-imperceptible seam with seam_gap=0, visible bump with seam_gap=10%.

### 11.3 nubnubbud — CR-10S Bowden confirmation

**nubnubbud** (CR-10S, Bowden extruder):
- Confirmed bug on 2.2.0 release, not on 2.2.0-RC
- Tried massive PA compensation: "Even with an entire millimeter of extra advance after a retract, it did nothing much"
- **Workaround that worked**: stayed on 2.2.0-RC until the revert landed

**Key takeaway**: PA tuning alone cannot fix the cf6d9c7 blob bug. The blob is a geometric/travel issue, not a pressure issue. (This doesn't matter for us on 2.3.2 since the revert is in place, but it's important context for anyone debugging scarf blobs.)

### 11.4 bdwilson — the late-breaking fix (2025-12-30)

**bdwilson**, the final comment before auto-close, reports success on BOTH a Snapmaker U1 (using Snorca, an OrcaSlicer fork) and a Bambu A1 using latest OrcaSlicer:

> "This works great on both my Snapmaker U1 and my Bambu A1 printers using latest Orca. The biggest change: Conditional scarf seams were enabled (`seam_slope_conditional = 1`)"

**The diff between his old and new profile**:
- Old: `seam_slope_conditional = 0` (scarf fires on everything, including short perimeters where it makes a bad blob)
- New: `seam_slope_conditional = 1` (scarf only fires on long, smooth perimeters)

**Our profile has `seam_slope_conditional: 1`**, so we benefit from this fix. ✅

### 11.5 pm0u — v2.3.0 revert confirmation

**pm0u**, 2025-05-01: printed identical test on 2.1.1 and 2.3.0, got identical results. **This confirms the cf6d9c7 revert is in 2.3.0** and therefore in 2.3.2.

**pm0u's 2.3.2 working profile** (2025-05-23, ABS, non-Bambu printer):
- max_volumetric_speed: 24 mm³/s
- outer_wall_speed: 200 mm/s (aggressive)
- scarf_joint_speed: 40% (= 80 mm/s)
- z_hop on retraction: 0.4 mm
- retract_when_changing_layer: ON
- wall_sequence: outer wall / inner wall (note: opposite of Adam L's recommendation)
- wall_loops: 4
- layer_height: 0.2 mm
- **Result**: "I'm not sure if I should hope for better but I thought this was a pretty good result"

### 11.6 What worked vs what didn't — consolidated list

Across all 80 comments of issue #7326, the consensus on workarounds:

**Things that helped some users**:
1. Downgrade to 2.1.1 or 2.2.0-RC (pre-cf6d9c7)
2. Wait for 2.3.0 (post-revert)
3. `seam_gap: 0` + negative extra length on restart (begna112)
4. Enable `seam_slope_conditional: 1` if coming from an old profile (bdwilson)
5. Wall order inner/outer or inner/outer/inner
6. Disable retract on layer change + set travel distance threshold high (shyblower)

**Things that did NOT help**:
1. PA tuning (nubnubbud — "did nothing much")
2. `wipe_on_loops: true` (made it worse)
3. `wipe_before_external_loop: true` (shyblower — "same result")
4. Z-hop adjustments alone
5. Flow ratio adjustments

---

## 12. Real-user empirical data — discussion #4325 + misc forum reports

### 12.1 Discussion #4325 (OrcaSlicer GitHub)

**khenderick** (2024-03-07): first successful use of scarf after PR #3839 merge. "layers 2+ appeared near perfect"; first layer's scarf looked odd ("starting ramp too thin").

**T4psi** (2024-03-20): tried `inner_wall_speed: 300` + `outer_wall_speed: 200`. Problem: "Scarf seam is not continuous...underextrusion(?) every time new seam line starts". **psiberfunk (Adam L) responded: "this is a no-go"** — those speeds are too high for scarf.

**evilDave** (2024-06-17): **Ender 3 V2 Neo, Bowden extruder, linear advance 0.5-0.6**. Result: "Bowden extruders work not so well with this feature" — blobs at start/end of scarf, underextrusion after. **Conclusion: Bowden setups are incompatible with scarf seams** due to the flow rate ramping requirement.

**wanderlust-vlad** (2024-03-14): Settings: 100% speed (60 mm/s), start height 0, 20 mm length, 10 steps. Problem: "Layer lines bumping out of model". Recommendation received: try whole-perimeter scarf with 0.1 mm start height.

**vgdh**: Recommended "start height 0.1 and seam gap = 0" — specifically for first-layer adhesion issues on scarf.

**markaudacity / cadriel** (late 2024-early 2025): OrcaSlicer 2.2.0 on X1C: "Can't get Orca to ever actually generate or print a scarf joint". Likely hitting the #7326 regression.

### 12.2 Voron forum (direct drive CoreXY community)

**stevereno30** (Voron 2.4, Clockwork → Orbiter, Dragon HF, ABS): already covered in Section 9.3. Key finding: **on high-flow hotends, PA tuning alone cannot eliminate both bulge and gap — overlap/seam_gap tuning is required**.

**JendaDH** (Voron 2.4, Phaetus Dragon UHF, Prusament ASA): already covered in Section 9.3. Fixed seam gap issues via PA + smooth_time tuning.

### 12.3 Prusa forum (PrusaSlicer 2.9 scarf)

**newtag** (Prusa MK3S, Prusament PETG, 0.2 mm quality profile, default scarf): **"The standard seam is now replaced by 2 smaller seams"** — scarf creates a visible end-of-overlap artifact. Two distinct seam points (start and end of overlap region) instead of one traditional seam.

This is a real expectation-setting finding: **"invisible" scarf seams are actually "two small seams" in the real world.** The goal isn't zero visible seam — it's that the two small seams blend in better than one big traditional seam.

**Neophyl** (Anycubic Neptune 3 Max, white PLA): tested Arachne and Classic, both scarf modes. Found `seam_gap` setting **"more useful" than scarf mode itself** — +5% adjustment reduced gaps more than enabling scarf did. Minimal difference from scarf overall on this geometry.

**Steve** (unspecified hardware, light beige filament, random seam position): **"The difference is amazing"** — removing dark seam holes visible on light-coloured filament.

**newtag** (MK3S, Prusament PLA, 2 perimeters, 0.15 mm layer height): no visible seam BUT **"vertical side is not vertical anymore with big deformation around the cylinder"** — indicates scarf can cause geometric deformation at thin walls / small features.

### 12.4 Venice3D comparison test

**Venice3D** (Prusa forum, 2024) tested 5 identical prints across PrusaSlicer 2.7.4, PrusaSlicer 2.8.0 alpha, and OrcaSlicer 2.0.0.

**Best result**: OrcaSlicer 2.0.0 with "Contour and hole" setting — **"much less visible"** than PrusaSlicer implementations.

Noted two distinct seam points (start and end of overlap region) — not one continuous hidden line.

### 12.5 Tony gervanne's planetary gear

**Tony gervanne** (Prusa forum): 1 kg planetary gear, 8 cogs.

- Before: 140 cm of seams requiring cutting and sanding for gear mesh
- After inner/outer/inner + scarf seams: **"about 90% better fit and regularity"**

Key insight: the inner/outer/inner wall order hides travel blobs internally rather than on the exterior surface.

### 12.6 Big_Cheese_Dog (V1E forum)

**Big_Cheese_Dog** (forum.v1e.com, OrcaSlicer seam settings thread) [^v1e-forum]:

**Key discovery**: Their outer shell was printing **twice as fast** as the inner shell. After reversing these speeds — outer wall **slower** than inner — seam quality improved substantially.

**Specific settings**: Scarf contour mode, seam gap 5%, outer wall slower than inner wall.

**Relevance for us**: our current config has `outer_wall_speed: 100` and `inner_wall_speed: 200`. **Inner is already faster than outer by 2×** — we're on the right side of this. ✅

### 12.7 PR #3839 comment thread (original implementation testing)

**FuryriderX** (first tester, PETG):
> "I had to set seam gap to 0 and disable wipe on loops and wipe on external loops."

Top sample with those settings: near-invisible seam. Middle sample (wipe enabled, gap 2%): visible artifact. Bottom (normal seam): visible seam.

**Noisyfox** directly confirmed:
> "seam gap should not matter when this is enabled, as I've already disabled the seam gap for external perimeter when scarf is used. If seam gap does make a difference in your test, then that's a bug."

This is where the "seam_gap should be 0 for scarf" consensus originated — the implementation author explicitly says it should not matter.

**MichaelJLew** proposed the fix for the phantom seam at retract point: move nozzle slightly inward before retraction so ooze blob is hidden.

### 12.8 Synthesis of real-user data

**The empirical consensus across 80+ documented tests**:

1. **Traditional seams**: seam_gap 0-15% is the recommended range; typical sweet spot 5-10% on a PA-tuned printer
2. **Scarf seams**: seam_gap SHOULD be 0 (or very low); Noisyfox's explicit statement, begna112's empirical confirmation
3. **Scarf speed**: 50-100 mm/s; higher = worse, lower = slower but OK
4. **Wall order**: inner-outer-inner wins for scarf quality (though pm0u used outer-first successfully)
5. **Wipe settings**: all three wipe flags OFF for scarf
6. **Conditional scarf**: ON (required for 2.3.x working profiles)
7. **Inner wall scarf**: ON helps with pressure cliff on large prints (from our own field notes)

---

## 13. PrusaSlicer scarf implementation — cross-slicer comparison

PrusaSlicer added scarf seams in **version 2.9.0 alpha**, more than a year after OrcaSlicer's PR #3839. The implementations are similar but not identical [^prusa-scarf-forum].

### 13.1 Source files and algorithm

Implementation lives in `src/libslic3r/GCode/SeamScarf.hpp` and `SeamScarf.cpp` [^prusa-seamscarf-cpp]:

```cpp
// Linear height ramp:
height_fraction = start_height + (distance / length) * (1.0 - start_height)
// Linear extrusion reduction:
e_fraction = 1.0 - distance / length
```

This is **identical in structure** to OrcaSlicer's approach but implemented separately. Both slicers converged on the same math independently.

### 13.2 PrusaSlicer scarf parameter names

From `src/libslic3r/PrintConfig.cpp`:

| Key | Default | Notes |
|---|---|---|
| `scarf_seam_placement` | `nowhere` (disabled) | Options: nowhere / contours / everywhere |
| `scarf_seam_only_on_smooth` | true | Default: only on smooth perimeters |
| `scarf_seam_start_height` | 0% | Fraction of layer height |
| `scarf_seam_entire_loop` | false | Extend to full perimeter |
| `scarf_seam_length` | 20 mm | Same as OrcaSlicer default |
| `scarf_seam_max_segment_length` | 1.0 mm | Min 0.15 mm |
| `scarf_seam_on_inner_perimeters` | false | |
| `seam_position` | aligned | Default: spAligned |

### 13.3 PrusaSlicer scarf bugs (for reference)

- **Issue #14640** [^prusa-issue-14640]: Negative seam gap + scarf causes nozzle to circle back over the finished loop. v2.9.2.
- **Issue #14278** [^prusa-issue-14278]: Scarf seam not placed at all in some configurations. v2.9.1.
- **Issue #14084** [^prusa-issue-14084]: Rear seam + scarf causes entire outer perimeter to print at wrong Z height on some layers. v2.9.0. **The reporter noted it does not happen when matching settings in OrcaSlicer 2.2.0** — suggesting OrcaSlicer's implementation is more robust.

**Takeaway**: PrusaSlicer's scarf is less mature. OrcaSlicer's is the better implementation for production use as of 2026.

### 13.4 Maturity comparison

- **OrcaSlicer's scarf is more mature** as of 2026 because it's been in stable releases for ~2 years and has Adam L's comprehensive tuning guide
- **PrusaSlicer's parameter set** is slightly different — fewer fine-tuning knobs, more opinionated defaults
- For our use case (OrcaSlicer 2.3.2), this is moot — we use OrcaSlicer's implementation

---

## 14. SuperSlicer's alternative approach (seam_notch, wipe_inside)

SuperSlicer [^superslicer-repo] is a fork of PrusaSlicer with **the most advanced seam controls of any slicer**, but it uses a fundamentally different approach: **notching** instead of scarf-ing.

### 14.1 The seam_notch concept

Instead of a sloped Z+E ramp, SuperSlicer **physically indents the toolpath** at the seam location — moving the nozzle slightly inward to create a small cavity. The seam blob still happens, but it lands **inside the wall**, not on the visible surface.

SuperSlicer description from the source:
> "It's sometimes very problematic to have a little bulge from the seam. This setting moves the seam inside the part, in a little cavity (for every seams in external perimeters, unless it's in an overhang)."

### 14.2 Relevant SuperSlicer settings (for reference only — not available in OrcaSlicer)

| Setting | Default | Description |
|---|---|---|
| `seam_gap` | **15%** (vs 10% in OrcaSlicer) | Default seam gap |
| `seam_gap_external` | 0 mm (disabled) | **Overrides `seam_gap` for external perimeters only** — lets you tune external and internal independently |
| `seam_notch_all` | 0 mm (disabled) | Size of the inward notch cavity |
| `seam_notch_inner` | 0 mm | Notch for round holes |
| `seam_notch_outer` | 0 mm | Notch for round external perimeters |
| `seam_notch_angle` | 250° (min 180) | If external angle exceeds this, no notch (not enough room geometrically) |
| `wipe_inside_end` | **true** (enabled) | Wipes nozzle inward after extruding external perimeter |
| `wipe_inside_depth` | 50% of perimeter width | How deep the inside wipe goes |
| `wipe_extra_perimeter` | 0 mm | Extra wipe distance along the loop before the inward wipe |

**Takeaway**: SuperSlicer's `seam_gap_external` (independent from `seam_gap`) is a feature that would be genuinely useful in OrcaSlicer — it would let us set `seam_gap: 0` for the external perimeter (for scarf) while keeping `seam_gap: 10%` for internal perimeters. No such split exists in OrcaSlicer as of 2.3.2.

**SuperSlicer has no scarf seam implementation.** Its approach to seam quality is notch + wipe_inside + aggressive PA tuning.

---

## 15. Bambu Studio, Cura — what the other slicers do

### 15.1 Bambu Studio

**Bambu Studio does not have scarf seams** — the feature is OrcaSlicer-specific. Bambu's primary approach to seam quality is: **high acceleration + input shaping makes seams less visible at speed.**

Looking at the BBL X1C 0.20 Standard profile [^bambu-bbl-x1c-profile]:
- `outer_wall_speed: 200` mm/s (standard), 350 mm/s (high flow)
- `inner_wall_speed: 300` mm/s (standard), 400 mm/s (high flow)
- `vertical_shell_speed: 80%` of outer wall
- `small_perimeter_speed: 50%`

**No scarf-specific settings.** No `seam_slope_type`. No `scarf_joint_*`.

**Implication**: Bambu's philosophy is "move fast enough that pressure dynamics settle quickly". Our Klipper-based hardware runs at much lower speeds (our outer wall is 100 mm/s vs Bambu's 200-350 mm/s), so this approach isn't transferable.

### 15.2 Cura

**Cura has no scarf seam or slope seam feature.** Cura's seam placement uses a different algorithm [^cura-fdmprinter-def]:

| Key | Default | Options |
|---|---|---|
| `z_seam_type` | `sharpest_corner` | back / shortest / random / sharpest_corner |
| `z_seam_position` | `back` | backleft/back/backright/right/frontright/front/frontleft/left |
| `z_seam_corner` | `z_seam_corner_inner` | none/inner/outer/any/weighted |
| `z_seam_relative` | false | Coordinates relative to part center |

**Cura's "coasting" feature** is the closest analog to OrcaSlicer's seam_gap — stops extrusion early and lets pressure coast through the line end:

- `coasting_enable`: false (default)
- `coasting_volume`: 0.064 mm³ (volume-based, not distance)
- `coasting_min_volume`: 0.8 mm³
- `coasting_speed`: 90% of print speed

**Takeaway**: Cura's `sharpest_corner` placement is generally equivalent to OrcaSlicer's corner-penalty algorithm (Section 3.3). Without scarf seams, Cura relies entirely on placement + coasting.

---

## 16. Community bundled profiles — per-profile comparison

Looking at what **actually-working bundled profiles** use for seam settings gives us real-world baselines [^orca-bundled-profiles].

### 16.1 Profiles with scarf actually enabled in OrcaSlicer 2.3.2

**Only three bundled profiles** in the OrcaSlicer 2.3.2 install actually have `seam_slope_type` set to anything other than `"none"`. All three are Creality Sermoon V1 profiles:

```json
// Creality Sermoon V1 0.20mm Standard + 0.16mm Optimal + 0.28mm Standard
{
  "seam_slope_type": "external",          // outer wall only, not hole
  "scarf_joint_flow_ratio": 0.95,         // slightly reduced
  "seam_gap": "6%",
  "seam_slope_min_length": 12,            // shorter than default 20
  "seam_slope_steps": 6,                  // fewer steps than default 10
  "outer_wall_line_width": 0.61,          // 152% of 0.4 nozzle (matches Adam L 150%!)
  "outer_wall_speed": 150,
  "outer_wall_acceleration": 4000,
  "outer_wall_jerk": 5,
  "precise_outer_wall": 1
}
```

**Observations**:
- Only outer wall (not holes) — less aggressive than our "all"
- Flow ratio at 95% — deliberately reduced
- min_length at 12 — shorter than default 20
- Only 6 steps — coarser ramp
- **outer wall width 0.61 mm = 152% of nozzle** — this matches Adam L's 150% recommendation almost exactly

**This is a real, shipping, vendor-maintained profile that validates Adam L's 150% outer wall width recommendation.**

### 16.2 Profiles with scarf fields set but scarf disabled (what "standard" looks like)

**Anycubic Kobra S1** (0.20mm Standard) — most complete modern reference:
```json
{
  "seam_slope_type": "none",
  "seam_slope_conditional": 1,            // conditional ON (ready for scarf)
  "seam_slope_min_length": 10,            // matches Bambu default, half code default
  "seam_slope_start_height": 0,
  "seam_slope_steps": 10,
  "seam_gap": "10%",
  "seam_position": "aligned",
  "scarf_angle_threshold": 155,
  "scarf_joint_flow_ratio": 1,
  "scarf_joint_speed": 35,                // absolute 35 mm/s — very slow!
  "scarf_overhang_threshold": "40%",
  "wall_generator": "classic",
  "wall_sequence": "outer wall/inner wall",
  "wipe_before_external_loop": 0,
  "wipe_on_loops": 0
}
```

**Key datapoint**: `scarf_joint_speed: 35` (absolute mm/s, not %). That's very slow — likely set for best possible quality when scarf IS enabled.

**Phrozen Arco 0.4** — similar "ready to enable" profile:
```json
{
  "seam_slope_type": "none",
  "seam_slope_conditional": 1,
  "seam_slope_min_length": 10,            // same as Anycubic
  "scarf_joint_speed": 35,                // also 35
  "outer_wall_speed": 200,
  "precise_outer_wall": 1
}
```

### 16.3 Klipper / Voron / VzBot bundled profiles

**Voron** (`fdm_process_voron_common.json`):
```json
{
  "seam_position": "aligned",
  "outer_wall_line_width": 0.4,
  "outer_wall_speed": 120,
  "outer_wall_acceleration": 3000,
  "outer_wall_jerk": 7,
  "wall_generator": "classic"
}
```
**No scarf settings at all.** Entirely default inheritance.

**VzBot** (`fdm_process_Vzbot_common.json`):
```json
{
  "seam_gap": "2%",                        // LOWEST of any bundled profile
  "seam_position": "nearest",
  "outer_wall_line_width": 0.42,
  "outer_wall_speed": 200,
  "outer_wall_acceleration": 5000,
  "wall_generator": "classic"
}
```
**No scarf.** `seam_gap: 2%` is the lowest of any bundled profile — closer to 0 than the 10% default. Suggests VzBot devs found tighter gaps better for traditional seams on high-speed prints.

**RatRig**:
```json
{
  "seam_position": "aligned",
  "outer_wall_line_width": 0.4,
  "outer_wall_speed": 120,
  "outer_wall_acceleration": 3000,
  "wall_generator": "classic"
}
```
Essentially identical to Voron. No scarf.

### 16.4 mjonuschat community Klipper profiles (GitHub)

https://github.com/mjonuschat/OrcaSlicer-Profiles — a maintained repo with Ellis-derived profiles for Voron and generic Klipper printers [^mjonuschat-profiles]:

**Voron PIF Quality**:
```json
{
  "seam_position": "back",
  "seam_gap": "15%",
  "wipe_on_loops": 1,                      // OPPOSITE of scarf recommendation
  "outer_wall_speed": 80,
  "outer_wall_line_width": "100%",
  "outer_wall_acceleration": 1000,         // very slow
  "precise_outer_wall": 1,
  "wall_generator": "classic"
}
```

**Voron PIF Fast**:
```json
{
  "seam_position": "back",
  "seam_gap": "15%",
  "wipe_on_loops": 1,
  "outer_wall_acceleration": 2000,
  "outer_wall_jerk": 5,
  "outer_wall_line_width": "100%",
  "precise_outer_wall": 1,
  "wall_generator": "classic"
}
```

**Annex Klipper**:
```json
{
  "seam_position": "back",
  "outer_wall_line_width": 0.5,
  "outer_wall_acceleration": 2000,
  "outer_wall_jerk": 5,
  "precise_outer_wall": 1,
  "wall_generator": "classic"
}
```

**Key takeaway**: The Ellis-derived Voron community profiles use `seam_gap: 15%`, `wipe_on_loops: 1`, **no scarf**, `seam_position: back`. This is the "aligned corner" camp, not the scarf camp. **Ellis's approach is that traditional seams can be made invisible enough with the right placement + PA tuning.**

### 16.5 Cross-profile comparison table

| Profile | Scarf type | Steps | Start h | Min L | Gap | Cond | Angle | Speed | Wall seq | Wipe loops | Wipe ext |
|---|---|---|---|---|---|---|---|---|---|---|---|
| **Creality Sermoon (scarf ON)** | external | 6 | 0 | 12 | 6% | no | 155 | default | outer/inner | 0 | 0 |
| **Adam L guide** | all | 10-20 | 0 or 50% | 20 | 0 | yes | 115 or 155 | ≤100 | inner-outer-inner | 0 | 0 |
| **Our 0.12mm Fine** | all | 20 | 0 | 20 | 0 | yes | 155 | 50% | inner-outer-inner | 0 | 0 |
| **Our 0.20mm PLA (CURRENT)** | all | 20 | 0 | 20 | **5%** | yes | 155 | 100% | inner-outer-inner | 0 | 0 |
| **BBL fdm_process_common** | none | 10 | 10% | 10 | — | no | 155 | default | — | 0 | 0 |
| **Voron bundled** | none | default | default | default | default | no | default | — | — | 0 | 0 |
| **VzBot bundled** | none | default | default | default | **2%** | no | default | — | — | 0 | 0 |
| **Anycubic Kobra S1** | none (ready) | 10 | 0 | 10 | 10% | yes | 155 | **35** | outer/inner | 0 | 0 |
| **mjonuschat Voron PIF** | none | default | default | default | **15%** | no | default | — | — | **1** | 0 |

**Conclusion**: our current PLA profile is fundamentally correct in shape. The only clearly-anomalous value is `seam_gap: 5%` — nobody else who uses scarf has 5%. Either 0 (for scarf quality) or 10%+ (for traditional seam buffer) makes more sense.

### 16.6 OrcaSlicer community / Ataraxia fork profiles

**Ataraxia-Mechanica/Unseam** [^unseam-repo] — while this is a post-processor, not a profile, it documents "required OrcaSlicer settings for Unseam": disable wipe_on_loops, ENABLE wipe_before_external_loop. This is the **opposite** of Adam L's recommendation — because Unseam is a post-processor that fixes the seam-gap artifact *after* the slicer emits gcode, it needs the raw wipe moves present in the gcode to modify them.

---

## 17. YouTube creator settings and Hackaday coverage

### 17.1 Teaching Tech (Michael Laws)

**Channel**: https://www.youtube.com/@TeachingTech
**Video**: "Scarf Joint Seams — how OrcaSlicer hides seams almost completely" (~March 2024)
**URL**: https://www.youtube.com/watch?v=vl0FT339jfc [^teaching-tech-video]

**Hardware**: Bambu X1C, 0.4 mm nozzle
**Filament**: Prusament Simply Green PLA (matches Adam L's test filament exactly)

**Teaching Tech's contribution**: he designed an alternative validation test model (Printables #784633 [^teaching-tech-test-model]) that covers many seam geometry types in a single print, complementing Adam L's guide. He's credited as the key collaborator validating the optimised settings.

**His specific settings (derived from video, linked guide, and Hackaday coverage)** [^hackaday-scarf]:

| Setting | Value |
|---|---|
| Wall/perimeter order | Inner/Outer/Inner |
| Outer wall speed | 75-100 mm/s (not above 150) |
| Scarf joint enabled | Contour + Hole |
| Conditional scarf | ON |
| Conditional angle threshold | 155° (production); 115° for testing |
| Scarf length | 20 mm |
| Scarf steps | 10 default (author uses 20) |
| **Scarf start height** | **0.1 mm for 0.2 mm layers = 50% of layer height** |
| Scarf flow ratio | 100% |
| Scarf joint speed | ≤100 mm/s |
| ERS | 300 mm³/s² on X1C |
| Outer wall width | 0.6 mm (150% of 0.4 nozzle) |
| Wipe on loops | **OFF** |
| Wipe on external loops | **OFF** |
| Wipe before external loop | **OFF** |
| Retract on layer change | **ON** |
| Wipe while retracting | **ON** |

**Critical finding**: Teaching Tech uses **seam_slope_start_height = 50%** (0.1 mm on 0.2 mm layers), not 10% or 0. This is higher than Bambu's 10% default. Higher start height = more clearance at the ramp start = less blob mechanism.

### 17.2 Maker's Muse (Angus Deveson)

**Channel**: https://www.youtube.com/@MakersMuse
**Video**: "Best seam settings — hide them completely! 3DP101" (2025)
**Source confirmation**: https://maker.forem.com/maker_youtube/makers-muse-best-seam-settings-hide-them-completely-3dp101-2hgg

**Content**: OrcaSlicer Z-seam settings covering Nearest, Aligned, Random, painted seam positioning, Spiral Vase Mode, and scarf seams. No specific numeric values extractable from description.

### 17.3 Other creators — no relevant content found

Searched but found no dedicated scarf seam content with numeric settings from:
- Uncle Jessy
- 3D Printing Nerd (Joel)
- My Tech Fun
- CHEP
- Slice Print Roleplay
- CNC Kitchen (Stefan) — has flow test content but no scarf video
- Thomas Sanladerer
- Vector3D
- Design Prototype Test
- Awesome Tech by Arindam

**Teaching Tech and Adam L are effectively the canonical YouTube-adjacent sources for scarf seam settings.**

### 17.4 Hackaday coverage

**Hackaday article** (2024-03-11): "Reducing Seams In FDM Prints With Scarf Joint Seams" [^hackaday-scarf]. Covers Teaching Tech's video and Adam L's guide. No new data beyond those sources, but validates them as the canonical references.

---

## 18. Tuning workflow (start-to-finish)

If you want to tune scarf seams from scratch on a new printer or new filament, here's the empirical process distilled from Adam L's testing + our own field experience.

### 18.1 Pre-requisite calibration

**Do not start scarf tuning until these are correct:**

1. **Pressure advance** — calibrated for the specific filament (per-filament, not per-printer)
2. **Flow rate / extrusion multiplier** — also per-filament
3. **Retraction settings** — appropriate for the extruder (Orbiter 2.5: 0.4-0.8 mm typical)
4. **Input shaper** — already done on our printer
5. **Bed levelling / Z offset** — first layer must be consistent
6. **Slicer version** — ensure you're on 2.3.0+ (post-cf6d9c7 revert)

If any of these are off, scarf seam testing is meaningless because the artefacts you'll see are from other causes.

### 18.2 Scarf tuning steps

1. **Download Adam L's TestSuite** from [Printables](https://www.printables.com/model/783313-better-seams-an-orca-slicer-guide-to-using-scarf-s) (TestSuite_OrcaSlicerOnly_V6.3MF)
2. **Set your filament profile** — temperature, flow, PA — for your specific filament
3. **Slice the test plate** with Adam L's recommended settings (the file ships with two optimised setting sets and one traditional reference)
4. **Print the test plate** on your machine
5. **Inspect the cylinders, double hexes, and curved pots** — compare scarf seam quality on the various test objects
6. **Pick the setting set that looks best**
7. **If overhangs are problematic**, switch to the second setting set (overhang-optimised)
8. **Tweak from there** based on the explanations in the guide

### 18.3 What to look for when inspecting test prints

- **Visible vertical line**: scarf ramp is too short, or speed is wrong, or pressure advance is off
- **Wall distortion** at the seam: speed is too high
- **"Smoosh" at inner walls** on overhangs: scarf inner walls is hurting more than helping — disable
- **Faint horizontal banding** along the scarf length: scarf steps may be too low
- **Over-extrusion bumps** in the seam region: check seam_gap is 0 for scarf, not 5-10%
- **Random blobs** elsewhere on the print: you've left `seam_position: random` on — switch to aligned
- **Blob at the start of scarf ramps on layers**: set `seam_slope_start_height: 10%` or `50%`
- **Seam landing at the wrong place**: Issue #13068 — seam is shifted by scarf length in 2.3.2

---

## 19. Post-processing options (Unseam, G-code editing)

### 19.1 Unseam — Python post-processor

**Repo**: https://github.com/Ataraxia-Mechanica/Unseam [^unseam-repo]

**What it does**: modifies the gcode of outer wall loops to "tuck in" the seam at both ends. The begin and end segments of each outer loop are inwardly offset by configurable amounts (`--tuck_begin`, `--tuck_end`, `--len_begin`, `--len_end`). Example defaults: `--tuck_begin 0.05 --len_begin 0.3 --tuck_end 0.1 --len_end 1`.

**Required OrcaSlicer settings for Unseam**: disable wipe on loops, **enable wipe before external loop**.

**Limitations**:
- Only 3 commits, minimal documentation, no releases
- Values require per-filament, per-layer-height, per-PA calibration
- Requires random seam placement and low layer heights for best results
- No reported community results specific to 2.3.x
- **Addresses the seam overlap region but does NOT address the travel-blob bug** — the blob occurs *before* the scarf begins, so tucking the seam end won't help with pre-seam nozzle-plunge artifacts

**Verdict**: useful supplement to traditional seams, but doesn't solve scarf-specific issues.

### 19.2 Manual G-code editing

**Technique demonstrated by @dodox1 in issue #7326**: comment out the standalone Z-move before the scarf start and instead merge it into the XYZ travel. Result: "So, there is Z movement simultaneously with X and Y movement, and this scar almost completely disappeared."

This is **not practical at scale** without a post-processor, but confirms the diagnosis. If the blob bug ever reappears, a Python script could identify `; SCARF_SEAM_START` markers in the gcode and modify the preceding Z move.

### 19.3 Klipper-side tricks

1. **Pressure Advance (PA) smoothing time**: A higher PA smoothing time (`smooth_time` in Klipper's `[extruder]` config, typically 0.04s) means the firmware averages the PA correction over a longer window, blunting the sharp pressure spike at the scarf start. Can reduce blob height marginally.

2. **Scarf speed setting**: Reducing `scarf_joint_speed` below 100 mm/s (try 50-75%) gives the hotend more time at consistent temperature and gives PA more time to stabilise.

3. **No Klipper macros exist** that specifically intercept and fix scarf seam issues at the firmware level — the mechanism is in the gcode path generation, not in motion execution.

### 19.4 Extrusion Rate Smoothing (ERS) for Klipper

Per the OrcaSlicer Wiki on advanced speed settings [^orca-wiki-speed]:

**ERS values by acceleration** (for 0.42 mm line width, 0.20 mm layer height):

| Acceleration | ERS starting range |
|---|---|
| 500 mm/s² | ~38 mm³/s² |
| 1000 mm/s² | ~76 mm³/s² |
| 2000 mm/s² | ~150 mm³/s² |
| 4000 mm/s² | ~300 mm³/s² |

**For our setup** at `machine_max_acceleration_extruding: 10000, 5000` (after today's cleanup, matches Klipper `max_accel`): ERS should be in the ~300-760 mm³/s² range per the formula. **Our current value is 300**, which matches Adam L's X1C value. This is validated by our own field notes (Case 5) as working on our specific hardware.

**Klipper segment length**: set to 1 (RPi has enough compute; only increase to 2-3 if "Timer too close" errors appear).

---

## 20. Downgrade considerations — is 2.1.1 the answer?

**Short answer**: no, because the cf6d9c7 blob bug **is not present in 2.3.2** (reverted in PR #8478 merged 2025-02-21). There's no version-regression benefit from downgrading.

**What you'd gain from downgrading to 2.1.1**:
- Confirmed working scarf seam reference version (if you can't find the regression on 2.3.2)

**What you'd lose**:
- The PR #8478 revert (but... you'd also not have the bug it reverted, so net zero)
- PR #9197: Arachne + scarf Z-hop fix (this MATTERS for us — we use Arachne)
- PR #10722: short travel deceleration before outer walls (minor quality improvement)
- PR #10803: Z-axis GCode handling improvements
- Various calibration fixes
- Profile format compatibility with newer filament/process presets
- 3MF path-traversal security fix (matters if you import external 3MFs)

**Recommendation**: **don't downgrade**. Fix the settings instead.

**If you want to test whether the current version has a different regression**: downgrade temporarily to 2.3.0 (first version with the revert) and compare. If 2.3.0 looks better than 2.3.2, there may be a 2.3.1 or 2.3.2 regression worth reporting. But this is extremely unlikely given the release notes.

---

## 21. Recommended settings for our hardware (final revision)

This is the **summary recommendation** combining all the research. For the canonical per-setting reference matrix with defaults and JSON keys, see [`quality_PLA.md`](quality_PLA.md). This section focuses on what we should **change** in our current profile.

### 21.1 Final recommendation table

| Setting | Current | Recommended | Confidence | Why |
|---|---|---|---|---|
| `seam_slope_type` | `all` | `all` | ✅ keep | Already correct per Adam L |
| `seam_slope_conditional` | `1` | `1` | ✅ keep | bdwilson's fix; already correct |
| `seam_slope_inner_walls` | `1` | `1` | ✅ keep | Field note Case 1 + Adam L |
| `seam_slope_steps` | `20` | `20` | ✅ keep | Aggressive smoothness, harmless |
| `seam_slope_min_length` | (inherits 20) | **20 explicit** | 🟢 Low risk | Lock in against future parent changes |
| `seam_slope_start_height` | (inherits 0) | **`10%` explicit** | 🟡 Medium-high | Bambu default; Teaching Tech uses 50%; 10% is safer start |
| `seam_slope_entire_loop` | (inherits false) | **`0` explicit** | 🟢 Low risk | Lock in |
| `scarf_angle_threshold` | (inherits 155) | **`155` explicit** | 🟢 Low risk | Lock in |
| `scarf_overhang_threshold` | (inherits 40%) | **`40%` explicit** | 🟢 Low risk | Lock in |
| `scarf_joint_speed` | (inherits 100) | **`75%`** | 🟡 Medium | Community sweet spot; Anycubic ships 35, Adam L says 75-100. 75% of our 100 mm/s outer wall = 75 mm/s, in the middle of Adam L's range |
| `scarf_joint_flow_ratio` | (inherits 1.0, dev mode) | (leave) | ✅ keep | Dev mode only; no evidence it helps |
| `seam_gap` | `5%` | **`0%`** | 🔴 **High priority** | Noisyfox + begna112 + FuryriderX all say 0 for scarf |
| `wipe_on_loops` | (inherits false) | **`0` explicit** | 🟢 Low risk | Lock in Adam L's "avoid" recommendation |
| `wipe_before_external_loop` | (inherits false) | **`0` explicit** | 🟢 Low risk | Lock in (also shyblower confirmed doesn't help with scarf) |
| `seam_position` | `aligned_back` | `aligned_back` | ✅ keep | Compatible with scarf; quality choice |
| `staggered_inner_seams` | `1` | `1` | ✅ keep | Adam L + quality benefit |
| `wall_sequence` | `inner-outer-inner wall` | same | ✅ keep | Adam L strongly recommends |
| `wall_generator` | `arachne` | same | ✅ keep | PR #9197 fixed the old Arachne+scarf bug in 2.3.1+ |
| `outer_wall_line_width` | `105%` | `105%` or `150%` (per-print choice) | ✅ keep 105% | Compromise; 150% is scarf-best but hits dimensional accuracy |
| `outer_wall_speed` | `100` mm/s | `100` | ✅ keep | Upper end of Adam L's 75-100 range |
| `precise_outer_wall` | `1` | `1` | ✅ keep | Already correct |

### 21.2 Prioritised change list (what to actually apply)

**🔴 High priority (will most likely fix the regression)**:
1. `seam_gap`: `5%` → `0%` — Noisyfox + begna112 + FuryriderX converging empirical evidence

**🟡 Medium priority (quality improvements)**:
2. Add `seam_slope_start_height: "10%"` explicit
3. Add `scarf_joint_speed: "75%"` explicit

**🟢 Low priority / cleanup (lock in inherited defaults)**:
4. Add `seam_slope_min_length: "20"` explicit
5. Add `scarf_angle_threshold: "155"` explicit
6. Add `scarf_overhang_threshold: "40%"` explicit
7. Add `seam_slope_entire_loop: "0"` explicit
8. Add `wipe_on_loops: "0"` explicit
9. Add `wipe_before_external_loop: "0"` explicit

**Recommendation**: apply all 9. The low-priority ones are zero-risk cleanups that make the profile self-documenting and robust against future parent profile changes. The high-priority one is the single most likely fix for the regression.

---

## 22. Normal seam tuning — the smallest gap without bulging

This section specifically addresses the user's request: "for the normal seam, I want the gap to be as small as possible without bulging out".

### 22.1 The PA-vs-seam_gap trade-off

Pressure advance and seam_gap are two levers for controlling the same physical phenomenon — the pressure dynamics at the loop start and end. They interact:

**Klipper rule of thumb** [^klipper-pa]:
- **Gap at seam → PA too high** (over-retracting at loop end, under-extruding at loop start)
- **Bump at seam → PA too low** (under-retracting at loop end, over-extruding)

**stevereno30's Voron 2.4 + Dragon HF + ABS data** (Section 9.3) shows this tension concretely: at PA 0.020 you get bulges, at PA 0.040 you get gaps, and there's no single value that fixes both. At that point the answer is to tune seam_gap to absorb the residual error.

**Our calibrated PA for Sunlu Matte PLA**: 0.045 (from field notes). This was tuned for clean seams before the recent input shaper changes. **It may need recalibration** — but we don't have evidence yet that it does.

### 22.2 seam_gap community ranges

From every source consulted:

| Source | Recommended seam_gap |
|---|---|
| OrcaSlicer wiki | 0-15% for PA-tuned printer |
| Noisyfox (implementation author) | 0 for scarf (applied at slope end) |
| begna112 (Sunlu PLA+ X1C) | 0 for scarf quality |
| VzBot bundled profile | 2% |
| Creality Sermoon scarf profile | 6% |
| OrcaSlicer code default | 10% |
| BBL fdm_process_common | 10% |
| Anycubic Kobra S1 | 10% |
| mjonuschat Voron PIF | 15% |
| SuperSlicer default | 15% |

**Range observed**: 0% to 15%. Median: ~8%.

### 22.3 Our specific recommendation for traditional-seam fallbacks

**The question**: with scarf enabled, when does traditional seam code actually fire?

**Answer**: only on perimeters that fail the conditional scarf check — specifically:
- Perimeters shorter than `seam_slope_min_length` (= 20 mm)
- Corners sharper than `scarf_angle_threshold` (= 155°)
- Overhangs above `scarf_overhang_threshold` (= 40%)

For these fallback traditional seams, the `seam_gap` value matters independently of the scarf setting.

**The dilemma**: setting `seam_gap: 0` optimises for scarf but may under-buffer traditional fallback seams on sharp corners. Setting `seam_gap: 10%` optimises for traditional but creates visible scarf clipping at the slope tail.

**The right answer for us**:
- Our profile is **scarf-dominant** — most features use scarf
- Traditional fallback is only on hard corners where visible seam is less of a concern (corners hide seams naturally)
- **Set `seam_gap: 0%` for scarf quality**; accept that sharp-corner fallbacks will have slightly less pressure buffering

This matches Adam L's approach and is consistent with begna112's direct empirical result on similar hardware.

### 22.4 If you ever print a traditional-seam-only profile

For a hypothetical profile with `seam_slope_type: none` (no scarf):

1. **Start at `seam_gap: 10%`** (OrcaSlicer wiki default, VzBot 2%, mjonuschat 15%)
2. **Inspect the print under raking light** at the seam location
3. **If bump visible**: PA too low OR seam_gap too high → reduce seam_gap by 2% OR increase PA
4. **If gap visible**: PA too high OR seam_gap too low → increase seam_gap by 2% OR reduce PA
5. **PA is the stronger lever** — tune PA first, seam_gap second
6. **Lower bound for seam_gap on well-tuned PA**: 2-5% (VzBot 2%, Creality Sermoon 6%)

**For Sunlu PLA Matte on our hardware** (Rapido 2 HF + Orbiter 2.5 + calibrated PA 0.045):
- Expected sweet spot: **5-8% seam_gap** for traditional seams
- Below 5%: risk of bumps if PA drifts
- Above 8%: risk of visible gaps

---

## 23. Open questions / things we couldn't pin down

- **Scarf behaviour at very small layer heights** (0.1 mm and below) — Adam L didn't test these
- **Scarf with 0.6 mm or 0.8 mm nozzles** — Adam L only tested 0.4 mm nozzle
- **Specific Rapido 2 HF + Orbiter 2.5 + Klipper + PLA scarf data** — not found in any public source. Everyone uses Bambu or Ender. We're genuinely in unexplored territory for this exact hardware combination.
- **Sunlu PLA Matte + scarf data** — none found. Matte PLA may need different settings because of filler viscosity, but no community data to confirm or deny
- **Per-filament-vendor PLA differences** — Adam L tested Prusament; other PLA brands might need slightly different scarf speed
- **Optimal ERS for our exact acceleration** — 300 works but is Adam L's X1C value; our own 10000 accel is 2.5× his so theoretically we could go higher. Field notes say 300 is fine.
- **TPU** — completely untested for scarf seams in any source I found
- **`seam_slope_start_height: 10%` vs `50%`** — Bambu default is 10%, Teaching Tech uses 50%. Which is better? No head-to-head data.

These are areas where if you ever care, you'd need to do your own testing — there's no community-tested answer to copy.

---

## 24. References

**Primary sources (read in full if going deeper)**:

[^adam-l-guide]: Adam L / @psiberfunk — "Better Seams: An Orca Slicer Guide to Using Scarf Seams" on Printables. The de facto community reference. 140+ experiments. PLA-specific. Tested on Bambu X1C. Includes downloadable `TestSuite_OrcaSlicerOnly_V6.3MF` test plate and Excel optimisation data sheet. <https://www.printables.com/model/783313-better-seams-an-orca-slicer-guide-to-using-scarf-s>

[^orca-pr-3839]: Scarf seams PR #3839 by Noisyfox / SoftFever — the original implementation, merged 2024-03-02. Technical implementation notes, default values, developer caveats, full comment thread with empirical testing from FuryriderX, igiannakas, MichaelJLew, and others. <https://github.com/SoftFever/OrcaSlicer/pull/3839>

[^orca-wiki-seam]: OrcaSlicer Seam Settings Wiki — official documentation for all seam settings, JSON keys, and recommended ranges. <https://github.com/SoftFever/OrcaSlicer/wiki/quality_settings_seam>

[^orca-issue-7326]: OrcaSlicer Issue #7326 — "Slicer creates an additional clearly perceptible and visible Z seam". 80-comment thread diagnosing and resolving the cf6d9c7 blob bug introduced in 2.2.0, reverted in 2.3.0 via PR #8478. Contains shyblower's mechanism analysis, begna112's empirical test on Sunlu PLA+, nubnubbud's CR-10S confirmation, and bdwilson's late-breaking finding about `seam_slope_conditional`. <https://github.com/SoftFever/OrcaSlicer/issues/7326>

[^orca-cf6d9c7]: Commit cf6d9c7 — the "Avoid collisions when moving Z down" commit by Fritz Webering that introduced the scarf blob regression in OrcaSlicer 2.2.0. Later reverted in PR #8478. <https://github.com/SoftFever/OrcaSlicer/commit/cf6d9c7>

[^orca-pr-8478]: PR #8478 — revert of cf6d9c7, merged 2025-02-21. Restores the pre-blob-bug scarf travel behaviour. Present in v2.3.0 and later. <https://github.com/SoftFever/OrcaSlicer/pull/8478>

[^orca-pr-9197]: PR #9197 — "Avoid unnecessary travel in scarf seam" — fixes Issue #9139 (Arachne + scarf = constant Z-hops and blobbing). Merged in 2025-04-13, included in v2.3.1-alpha and later. Important for Arachne users (like us). <https://github.com/SoftFever/OrcaSlicer/pull/9197>

[^orca-pr-10722]: PR #10722 — "Reduce artifacts from short travel moves before external perimeters". Short travel moves before outer wall printing now use outer wall acceleration instead of travel acceleration. Merged in v2.3.2-beta. Helps with seam-adjacent vibration artifacts. <https://github.com/OrcaSlicer/OrcaSlicer/pull/10722>

[^orca-issue-13068]: OrcaSlicer Issue #13068 — "When scarf is enabled, seam placement shifts location by one scarf length from expected position". v2.3.2 open issue. Seam preview and actual seam landing location differ by one scarf length. <https://github.com/OrcaSlicer/OrcaSlicer/issues/13068>

[^orca-issue-12049]: OrcaSlicer Issue #12049 — "Scarf joint seam drastically reduces quality of smooth surfaces for objects with protruding parts". v2.3.0-2.3.1. Affects models with secondary intersecting features. <https://github.com/OrcaSlicer/OrcaSlicer/issues/12049>

[^orca-issue-12270]: OrcaSlicer Issue #12270 — "Conditional scarf overhang threshold not being respected". Scarf fires on overhanging perimeters regardless of threshold. v2.3.1. <https://github.com/OrcaSlicer/OrcaSlicer/issues/12270>

[^orca-issue-7892]: OrcaSlicer Issue #7892 — "Adaptive PA + Scarf seam incompatibility". Closed NOT_PLANNED. Adaptive PA does not recalculate during scarf slope. <https://github.com/SoftFever/OrcaSlicer/issues/7892>

[^orca-printconfig]: OrcaSlicer `PrintConfig.cpp` — source code location for all scarf/seam setting code defaults. <https://raw.githubusercontent.com/OrcaSlicer/OrcaSlicer/main/src/libslic3r/PrintConfig.cpp>

[^orca-seam-placer-cpp]: OrcaSlicer `SeamPlacer.cpp` — source code for the corner-penalty seam placement algorithm. <https://raw.githubusercontent.com/OrcaSlicer/OrcaSlicer/main/src/libslic3r/GCode/SeamPlacer.cpp>

[^orca-bbl-fdm-common]: OrcaSlicer Bambu Lab bundled `fdm_process_common.json` — BBL base profile that sets `seam_slope_min_length: 10`, `seam_slope_start_height: 10%`. Source of Bambu-specific defaults that Klipper-based profiles don't inherit. <https://raw.githubusercontent.com/OrcaSlicer/OrcaSlicer/main/resources/profiles/BBL/process/fdm_process_common.json>

[^orca-2-0-release]: OrcaSlicer v2.0.0 release notes — documents `scarf_joint_flow_ratio` move to developer mode only. Links to Adam L's Printables guide as the definitive reference. <https://github.com/OrcaSlicer/OrcaSlicer/releases/tag/v2.0.0>

[^orca-discussion-4325]: OrcaSlicer Discussion #4325 — "Scarf joint seams" community thread. Contains user reports from khenderick, T4psi, evilDave, wanderlust-vlad, vgdh, markaudacity, cadriel, and Adam L's responses. <https://github.com/SoftFever/OrcaSlicer/discussions/4325>

[^orca-wiki-line-width]: OrcaSlicer Line Width Wiki. Recommends 100% of nozzle for outer wall (dimensional accuracy), 105-150% for other features. Contrasts with Adam L's 150% for scarf quality. <https://github.com/SoftFever/OrcaSlicer/wiki/quality_settings_line_width>

[^orca-wiki-speed]: OrcaSlicer Speed Settings Advanced Wiki — includes ERS (extrusion rate smoothing) tables by acceleration. <https://github.com/OrcaSlicer/OrcaSlicer/wiki/speed_settings_advanced>

[^orca-pr-4317]: PR #4317 by SoftFever — added the conditional scarf joint (`seam_slope_conditional`) and `scarf_joint_speed` slowdown. Sets default `scarf_angle_threshold` to 155°. <https://github.com/SoftFever/OrcaSlicer/pull/4317>

[^orca-pr-4725]: PR #4725 — "Add overhang threshold for scarf joint seam". Introduced `scarf_overhang_threshold` in v2.0+. <https://github.com/SoftFever/OrcaSlicer/pull/4725>

[^orca-pr-10255]: PR #10255 — "Introduce a new seam alignment option: Aligned back". Added `aligned_back` seam position option. <https://github.com/OrcaSlicer/OrcaSlicer/pull/10255>

[^orca-issue-9139]: OrcaSlicer Issue #9139 — "Arachne + scarf seam = constant Z-hops and blobbing". v2.3.0 bug, fixed by PR #9197. <https://github.com/SoftFever/OrcaSlicer/issues/9139>

[^orca-issue-5465]: OrcaSlicer Issue #5465 — "Scarf seam flow rate drops below volumetric limit". Closed NOT_PLANNED. Particularly affects "entire loop" mode. <https://github.com/SoftFever/OrcaSlicer/issues/5465>

[^orca-issue-8391]: OrcaSlicer Issue #8391 — "Painted seam + scarf = ugly seam at painted location". Workaround: remove painted seam. <https://github.com/SoftFever/OrcaSlicer/issues/8391>

[^orca-issue-7476]: OrcaSlicer Issue #7476 — "Painted seam position offset by scarf length". Seam appears at paint location + scarf length. <https://github.com/SoftFever/OrcaSlicer/issues/7476>

[^orca-issue-6378]: OrcaSlicer Issue #6378 — "Wide seam gap regression with multi-object prints". Closed NOT_PLANNED. Affects 2+ STL simultaneous prints. <https://github.com/SoftFever/OrcaSlicer/issues/6378>

[^orca-issue-11743]: OrcaSlicer Issue #11743 — "Horizontal scarf proposal". Not implemented. Proposes horizontal overlap instead of vertical Z-ramp. <https://github.com/SoftFever/OrcaSlicer/issues/11743>

[^orca-issue-11834]: OrcaSlicer Issue #11834 — "Unseam / tuck-in feature request". References the Ataraxia-Mechanica/Unseam post-processor. <https://github.com/SoftFever/OrcaSlicer/issues/11834>

[^orca-pr-12974]: PR #12974 — "Add Precise Seam placement feature". Open as of 2026-04. Volume-based seam modifier with 6 types. <https://github.com/OrcaSlicer/OrcaSlicer/pull/12974>

[^orca-discussion-1389]: OrcaSlicer Discussion #1389 — "Iron only top and bottom layer and/or choose ironing layers". Related to seam painting. <https://github.com/SoftFever/OrcaSlicer/discussions/1389>

[^orca-discussion-7540]: OrcaSlicer Discussion #7540 — "Persistent blobs after upgrade from 1.8 to 2.2". User reports consistent with #7326 cf6d9c7 regression. <https://github.com/OrcaSlicer/OrcaSlicer/discussions/7540>

[^orca-bundled-profiles]: OrcaSlicer bundled profile JSONs at `resources/profiles/*/process/*.json` in the GitHub main branch. Includes Creality (only scarf-enabled bundled profiles), Anycubic Kobra S1, Phrozen Arco, Voron, VzBot, RatRig, BBL. <https://github.com/SoftFever/OrcaSlicer/tree/main/resources/profiles>

**Secondary sources**:

[^obico-orca-seam]: Obico blog — "OrcaSlicer Seam Settings". Clear explainer for each setting, less depth than Adam L. <https://www.obico.io/blog/orcaslicer-seam-settings/>

[^all3dp-seam-gap]: All3DP — "Orca Slicer: Seam Gap tutorial". Beginner-oriented overview. <https://all3dp.com/2/orca-slicer-seam-gap-tutorial/>

[^hackaday-scarf]: Hackaday (2024-03-11) — "Reducing Seams In FDM Prints With Scarf Joint Seams". Covers Teaching Tech's video and Adam L's guide. <https://hackaday.com/2024/03/11/reducing-seams-in-fdm-prints-with-scarf-joint-seams/>

[^teaching-tech-video]: Teaching Tech YouTube — "Scarf Joint Seams — how OrcaSlicer hides seams almost completely" (2024-03). <https://www.youtube.com/watch?v=vl0FT339jfc>

[^teaching-tech-test-model]: Teaching Tech — Scarf Seam Test Model on Printables #784633. Alternative validation test complementing Adam L's test suite. <https://www.printables.com/model/784633-scarf-seam-test-model>

[^prusa-scarf-forum]: Prusa Forum — "PrusaSlicer 2.9.0 alpha1: how to use scarf seam". PrusaSlicer community testing of their independent scarf implementation. <https://forum.prusa3d.com/forum/prusaslicer/prusaslicer-2-9-0-alpha1-how-to-use-scarf-seam/>

[^prusa-scarf-comparison]: Prusa Forum — "Scarf seams" comparison thread, Venice3D's 5-print comparison of PrusaSlicer 2.7.4, 2.8.0-alpha, and OrcaSlicer 2.0.0. Found OrcaSlicer "much less visible". <https://forum.prusa3d.com/forum/english-forum-original-prusa-i3-mk4-general-discussion-announcements-and-releases/scarf-seams/>

[^prusa-mk4-is-seam]: Prusa Forum — "MK4 IS seam scar". D3XT3R identifies PrusaSlicer 2.7.1 as the source of a seam regression independent of firmware. <https://forum.prusa3d.com/forum/input-shaping/mk4-is-seam-scar/>

[^prusa-issue-11621]: PrusaSlicer Issue #11621 — MichaelJLew's original "scarf seam" proposal. The discussion where the term was coined. <https://github.com/prusa3d/PrusaSlicer/issues/11621>

[^prusa-seamscarf-cpp]: PrusaSlicer source — `SeamScarf.hpp` and `SeamScarf.cpp`. Implementation of PrusaSlicer's scarf seams. <https://github.com/prusa3d/PrusaSlicer/blob/main/src/libslic3r/GCode/SeamScarf.cpp>

[^prusa-issue-14640]: PrusaSlicer Issue #14640 — Negative seam gap + scarf causes nozzle to circle back. v2.9.2. <https://github.com/prusa3d/PrusaSlicer/issues/14640>

[^prusa-issue-14278]: PrusaSlicer Issue #14278 — Scarf not placed in some configurations. v2.9.1. <https://github.com/prusa3d/PrusaSlicer/issues/14278>

[^prusa-issue-14084]: PrusaSlicer Issue #14084 — Rear seam + scarf causes wrong Z height on some layers. v2.9.0. Reporter notes OrcaSlicer 2.2.0 handles it correctly. <https://github.com/prusa3d/PrusaSlicer/issues/14084>

[^superslicer-repo]: SuperSlicer repository — advanced fork of PrusaSlicer with seam_notch, wipe_inside, and seam_gap_external features. <https://github.com/supermerill/SuperSlicer>

[^bambu-bbl-x1c-profile]: Bambu Studio BBL X1C 0.20mm Standard bundled profile JSON. <https://raw.githubusercontent.com/bambulab/BambuStudio/master/resources/profiles/BBL/process/0.20%20Standard%20%40BBL%20X1C.json>

[^cura-fdmprinter-def]: Cura `fdmprinter.def.json` — base printer definition with z_seam_type and coasting settings. <https://raw.githubusercontent.com/Ultimaker/Cura/master/resources/definitions/fdmprinter.def.json>

[^mjonuschat-profiles]: mjonuschat/OrcaSlicer-Profiles — maintained community profiles for Voron and generic Klipper printers, derived from Ellis SuperSlicer profiles. Uses `seam_gap: 15%`, no scarf. <https://github.com/mjonuschat/OrcaSlicer-Profiles>

[^voron-stevereno]: Voron Forum — stevereno30's "Question — Square corner bulges / low PA corner gapping" thread. Voron 2.4 + Dragon HF + ABS PA tuning data showing the bulge-vs-gap trade-off. <https://forum.vorondesign.com/threads/question-square-corner-bulges-low-pa-corner-gapping.345/>

[^voron-jendadh]: Same Voron forum thread — JendaDH's Voron 2.4 + Phaetus Dragon UHF + Prusament ASA data. PA range 0.038-0.052, extrusion multiplier 93-95%, 265°C print temp.

[^unseam-repo]: Ataraxia-Mechanica/Unseam — Python gcode post-processor that tucks seam ends inward. Not a scarf replacement; supplements traditional seams. <https://github.com/Ataraxia-Mechanica/Unseam>

[^v1e-forum]: V1E Forum — "Orca Slicer seam settings" thread. Big_Cheese_Dog's finding that inner-faster-than-outer speed ratio improved scarf quality. <https://forum.v1e.com/t/orca-slicer-seam-settings/52339>

[^klipper-pa]: Klipper Pressure Advance documentation. Rule of thumb: "gap at seam → PA too high, bump at seam → PA too low." <https://www.klipper3d.org/Pressure_Advance.html>

[^orca-scarf-profile-makerworld]: Adam L — "Scarf Seam Profile — Does It Get Any Better?" on MakerWorld. Later refinement profile. <https://makerworld.com/en/models/422555-scarf-seam-profile-does-it-get-any-better>

[^orca-better-seams-makerworld]: Adam L — "Better Seams" guide mirrored on MakerWorld. <https://makerworld.com/en/models/211686-better-seams-orca-slicer-guide-to-scarf-seams>

[^obico-seam-settings-main]: Obico — "OrcaSlicer Seam Settings" comprehensive walkthrough. <https://www.obico.io/blog/orcaslicer-seam-settings/>

[^minimal3dp-ers]: Minimal3DP — "Extrusion Rate Smoothing in OrcaSlicer". Technical explanation of ERS and its interaction with scarf. <https://minimal3dp.com/blog/extrusion-rate-smoothing-in-orcaslicer/>

[^orca-issue-7208]: OrcaSlicer Issue #7191 / PR #7208 — the original "nozzle colliding with walls" issue that commit cf6d9c7 attempted to fix. Context for the 2.2.0 regression. <https://github.com/SoftFever/OrcaSlicer/issues/7191>

[^orca-issue-10245]: OrcaSlicer Issue #10245 — "Unwanted extrusion over top layer causes pressure build-up and visible blobs on seam start". Closed stale. Related to general seam blob issues. <https://github.com/SoftFever/OrcaSlicer/issues/10245>

[^orca-pr-10803]: OrcaSlicer PR #10803 — "Enhance GCode handling for Z-axis movements". Merged in v2.3.1. Minor motion planner improvement. <https://github.com/OrcaSlicer/OrcaSlicer/pull/10803>

[^orca-pr-12028]: OrcaSlicer PR #12028 — "Fix aligned back seam positioning for mirrored objects". Merged in v2.3.2-beta. <https://github.com/OrcaSlicer/OrcaSlicer/pull/12028>

[^orca-pr-11923]: OrcaSlicer PR #11923 — "Dual seam fuzzy painted" fix. Merged in v2.3.2-rc. <https://github.com/OrcaSlicer/OrcaSlicer/pull/11923>

[^orca-issue-3450]: OrcaSlicer Issue #3450 — `wipe_before_external_loop` blob issue. Fixed by PR #3616 in early 2024. <https://github.com/SoftFever/OrcaSlicer/issues/3450>

**Cross-references in this repo**:

- [`quality_PLA.md`](quality_PLA.md) — per-setting matrix for the Quality tab. The `seam` section there is the canonical "what we have set" reference
- [`seam_diagnosis_field_notes.md`](seam_diagnosis_field_notes.md) — empirical counterpart to this theoretical deep-dive. Real-print case studies from our own hardware
- [`../filament/sunlu_pla_matte.md`](../filament/sunlu_pla_matte.md) — Sunlu PLA Matte filament profile including PA calibrated to 0.045
- [`../printer/extruder.md`](../printer/extruder.md) — printer-side retraction/wipe settings (Z hop spiral lift, retract_when_changing_layer, wipe)
- [`../printer/motion_ability.md`](../printer/motion_ability.md) — motion limits relevant to ERS calculation
- [`README.md`](README.md) — index for all process tab documents
