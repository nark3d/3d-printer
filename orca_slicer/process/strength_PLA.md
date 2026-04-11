# OrcaSlicer Strength tab settings — PLA quality profile

> One file per tab. See [`README.md`](README.md) for the index of all tab documents and shared hardware context (printer, toolhead, hotend, extruder, motion limits, material, goal).
>
> Working document. For each OrcaSlicer Strength tab setting: the OrcaSlicer default, the *researched* value (community references, vendor docs, GitHub discussions — with footnote citations), our current value, and comments documenting reasoning behind any deviation.

---

## Strength tab

### Walls

| Setting | OrcaSlicer default | Researched value | Our value | Comments |
|---|---|---|---|---|
| Wall loops | 3 | **4** (functional/quality), 2 (decorative only) [^wall-loops] | **4** | Community consensus: 4 walls gives better strength-per-weight than adding infill density. **DECISION: no change** — override is correct. |
| Alternate extra wall | `false` | `false` *(no community consensus)* | `false` *(inherited)* | Adds an extra outer wall every other layer. Niche feature. **DECISION: no change** — default is correct. |

### Top/bottom shells

| Setting | OrcaSlicer default | Researched value | Our value | Comments |
|---|---|---|---|---|
| Top shell layers | 4 | 3-5 [^shells] | **5** | Override 5 is slightly conservative end of community range. Pairs well with ironing on topmost layer. **DECISION: no change** — override is correct. |
| Bottom shell layers | 3 | 3-5 [^shells] | **5** | Same — extra protection for functional baseplates. **DECISION: no change** — override is correct. |
| Top shell thickness | 0.8 mm | 0.8 mm *(default)* [^shells] | 0.8 mm *(inherited)* | Safety floor (increases layer count if `layers × height < thickness`). At our 0.2 mm × 5 = 1.0 mm, the floor is irrelevant. **DECISION: no change**. |
| Bottom shell thickness | 0 mm | 0 mm *(default)* [^shells] | 0 mm *(inherited)* | 0 means "use layer count, no floor". We use 5 layers explicitly. **DECISION: no change**. |
| Top surface pattern | `monotonicline` | `monotonic` or `monotonicline` [^patterns] | `monotonicline` *(inherited)* | Both give cleanest top surface. `monotonicline` is slightly faster. With ironing, either works. **DECISION: no change** — default is correct. |
| Bottom surface pattern | `monotonic` | `monotonic` [^patterns] | `monotonic` *(inherited)* | Standard for clean bottom surface. **DECISION: no change** — default is correct. |
| Internal solid infill pattern | (varies, often `monotonic`) | `monotonic` [^patterns] | (default) | Internal (non-visible) solid layers. **DECISION: no change** — default is correct. |

### Infill

| Setting | OrcaSlicer default | Researched value | Our value | Comments |
|---|---|---|---|---|
| Sparse infill density | 15 % | **15-20 %** for functional, 10 % for decorative [^infill] | 15 % *(inherited)* | Above 20% gives diminishing strength returns per gram; wall_loops 4 already provides the strength. **DECISION: no change** — default 15% is correct. |
| Sparse infill pattern | `crosshatch` | **`gyroid`** (best strength-to-weight) [^patterns] [^infill-patterns] | **`gyroid`** | Gyroid is the community consensus for best 3-axis strength-to-weight including Z. **DECISION: no change** — override is correct. |
| Lightning infill (if enabled) | `false` | `false` *(quality goal)* | (default) | Minimal-material support-only pattern — almost zero structural strength. **DECISION: no change** — off for quality goal. |
| Length of infill anchor | 400 % | 400 % *(default)* [^infill-anchor] | (default) | Length over which infill connects to perimeter walls. **DECISION: no change** — default is well-tuned. |
| Maximum length of infill anchor | (default ~12 mm) | default *(works for most)* [^infill-anchor] | (default) | Maximum anchor length. **DECISION: no change** — default is correct. |
| Infill direction | 45° | 45° *(default)* | 45° *(inherited)* | 45° spreads loads along both X and Y. **DECISION: no change** — default is correct. |
| Infill rotation angle | 0° | 0° *(default)* | (default) | Per-layer rotation of infill direction. Irrelevant for gyroid (which rotates internally). **DECISION: no change** — default is correct. |
| Apply gap fill | `everywhere` | `everywhere` *(default for quality)* | **`everywhere`** | Fills all gaps where a perimeter is too narrow. **DECISION: no change** — override matches default, quality-first. |
| Filter out tiny gaps | `false` | `false` *(default)* | (default) | Would skip gap-fill for very small gaps. **DECISION: no change** — default off is fine. |

### Advanced

| Setting | OrcaSlicer default | Researched value | Our value | Comments |
|---|---|---|---|---|
| Infill/wall overlap | 25 % | **10-15 %** (Orca recommends) [^infill-wall-overlap] | **15 %** | Default 25% causes visible bumps on inside walls. Our 15% is at top of the recommended 10-15% range — good bonding without over-extrusion. **DECISION: no change** — override is correct. |
| Infill combination | `false` | `false` *(default)* [^infill] | `false` *(inherited)* | Combines sparse infill layers for speed but causes visible step lines. **DECISION: no change** — off for quality goal. |
| Detect narrow internal solid infill | `true` | `true` *(default)* | (default) | Adds extra solid infill in narrow corners where sparse would leave weak spots. **DECISION: no change** — default on is correct. |
| Ensure vertical shell thickness | (Quality tab) | **`moderate`** [^vertical-shell] | (inherited — see Quality tab) | Duplicate of Quality tab setting. **DECISION: CHANGE to `moderate`** (already flagged in Quality tab review — same setting). |
| Interface shells | `false` | `false` *(default)* | `false` *(inherited)* | Multimaterial-only feature. **DECISION: no change** — N/A for single-material. |
| Minimum sparse infill area | 15 mm² | 15 mm² *(default)* | 15 mm² *(inherited)* | Areas smaller than this are filled solid (too small for sparse to anchor). **DECISION: no change** — default is well-tuned. |

---

## Summary of recommended Strength tab changes from current overrides

The Strength tab analysis identifies these settings:

### Confirmed correct (no change needed)

Our current overrides on the Strength tab are well-supported by community consensus:

- `wall_loops: 4` — community confirms 4 walls is better strength-per-gram than adding infill density
- `top_shell_layers: 5` — within recommended 3-5 range, on the conservative side for quality
- `bottom_shell_layers: 5` — same
- `sparse_infill_pattern: gyroid` — best general-purpose strength-to-weight
- `infill_wall_overlap: 15%` — at the top of OrcaSlicer's recommended 10-15 % range
- `gap_fill_target: everywhere` — quality choice, matches default

### No changes recommended

The Strength tab is already well-tuned. Unlike the Quality tab (where I found 8+ recommended changes), the Strength tab's defaults plus our existing overrides are essentially correct. **No new changes from this research pass.**

The only setting that interacts with the Strength tab from elsewhere is `ensure_vertical_shell_thickness` which is recommended to be set to `moderate` per the [Quality tab analysis](quality_PLA.md) — but that's a Quality-tab change, not a Strength-tab one.

---

## References

[^wall-loops]: OrcaSlicer Walls Wiki and community guidance — 2 walls for decorative, 3 for functional default, 4+ provides better strength-to-weight than adding infill density. <https://github.com/SoftFever/OrcaSlicer/wiki/strength_settings_walls> · <https://www.orcaslicer.com/wiki/print_settings/strength/strength_settings_walls>

[^shells]: OrcaSlicer Top and Bottom Shells Wiki — minimum 3 shell layers recommended, common range 3-5 depending on quality and strength requirements. Shell thickness setting acts as a floor: if `shell_layers × layer_height < shell_thickness`, OrcaSlicer increases the layer count. <https://github.com/SoftFever/OrcaSlicer/wiki/strength_settings_top_bottom_shells>

[^patterns]: OrcaSlicer Patterns Wiki and Bambu Lab Wiki on infill patterns. **Gyroid**: rotating sine wave, excellent strength-to-weight ratio, uniform 3-axis stress distribution including Z. **Honeycomb**: high strength but ~25 % more material. **Cubic**: fast print with decent strength. **Crosshatch**: standard default, fastest, weaker than gyroid in Z. For PLA quality goal, gyroid is the standard recommendation. <https://github.com/SoftFever/OrcaSlicer/wiki/strength_settings_patterns> · <https://wiki.bambulab.com/en/software/bambu-studio/fill-patterns>

[^infill]: OrcaSlicer Infill Wiki — sparse infill density recommendations and infill combination behaviour. Default 15 % sparse infill is the standard "quality plus reasonable strength" choice. Going above 20 % gives diminishing strength returns per gram. <https://github.com/SoftFever/OrcaSlicer/wiki/strength_settings_infill>

[^infill-patterns]: All3DP: "The Strongest Infill Patterns for Maximum 3D Print Strength" — community comparison of infill pattern strength. Confirms gyroid as the best general-purpose strength-to-weight choice. <https://all3dp.com/2/strongest-infill-pattern/>

[^infill-anchor]: OrcaSlicer Infill Wiki on `sparse_infill_anchor` and `sparse_infill_anchor_max`. Default 400 % anchor length. Setting to 0 reverts to the legacy algorithm. Useful for specialised cases (e.g. avoiding artefacts on open-back features) but default is correct for general use. <https://github.com/SoftFever/OrcaSlicer/wiki/strength_settings_infill>

[^infill-wall-overlap]: OrcaSlicer Infill Wiki — recommends `infill_wall_overlap` of 10-15 % to "minimise potential over-extrusion and accumulation of material resulting in rough surfaces". The default 25 % causes visible bumps on inside walls. <https://github.com/SoftFever/OrcaSlicer/wiki/strength_settings_infill>

[^vertical-shell]: See [`quality_PLA.md`](quality_PLA.md) `[^vertical-shell]` — multiple GitHub issues report `ensure_vertical_shell_thickness = all` causes a visible hull line artefact. `moderate` is the safer middle ground.
