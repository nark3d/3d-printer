# OrcaSlicer Others tab settings — PLA quality profile

> One file per tab. See [`README.md`](README.md) for the index of all tab documents and shared hardware context.
>
> The Others tab covers skirt/brim, special print modes (spiral vase, fuzzy skin, sequence), G-code output options, post-processing scripts, and notes. Most of these are **use-case-dependent rather than quality-dependent** — there's no single "right value" for skirt_loops because the right value depends on whether you're priming the nozzle, doing first-layer leveling, or just skipping it entirely.
>
> Working document. Same column structure as the other tabs.

---

## Others tab

### Skirt

| Setting | OrcaSlicer default | Researched value | Our value | Comments |
|---|---|---|---|---|
| Skirt loops | 0 | **1-3** [^skirt-brim] | (default 0) *(inherited)* | Our `PRIME_LINE` macro primes the nozzle separately and Beacon verifies Z; skirt is redundant for us. **DECISION: no change** — default 0 is correct. |
| Skirt distance from object | 2 mm | 5 mm [^skirt-brim] | (default 2) *(inherited)* | Only matters if skirt is on. **DECISION: no change** — moot. |
| Skirt height | 3 layers | 1-3 layers [^skirt-brim] | (default 3) *(inherited)* | Only matters if skirt is on. **DECISION: no change** — moot. |
| Minimum skirt length | 4 mm | 4 mm *(default)* | (default 4) *(inherited)* | Only matters if skirt is on. **DECISION: no change** — default tuned. |

### Brim

| Setting | OrcaSlicer default | Researched value | Our value | Comments |
|---|---|---|---|---|
| Brim type | `auto_brim` | `no_brim` for PLA on textured PEI [^skirt-brim] | **`no_brim`** | Textured PEI + Beacon + tuned bed temp = no brim needed. Per-model brim can still be added in prepare view. **DECISION: no change** — override is correct. |
| Brim width | 5 mm | 5-8 mm (when used) [^skirt-brim] | (default 5) *(inherited)* | Only matters if brim is enabled. **DECISION: no change** — moot. |
| Brim object gap | 0.1 mm | 0-0.12 mm [^skirt-brim] | (default 0.1) *(inherited)* | Only matters if brim is enabled. **DECISION: no change** — moot. |
| Inner brim only | `false` | `false` *(default)* | (default) | Only relevant for parts with lifting protrusions. **DECISION: no change** — default off. |

### Special mode

| Setting | OrcaSlicer default | Researched value | Our value | Comments |
|---|---|---|---|---|
| Slicing mode | `regular` | `regular` *(default)* | (default) | Options: regular, even-odd, close holes. Non-default modes are for edge cases. **DECISION: no change** — default is correct. |
| Print sequence | `by layer` | **`by layer`** [^print-sequence] | `by layer` *(inherited)* | Standard mode, supports filament change at layer. **DECISION: no change** — default is correct. |
| Spiral vase | `false` | `false` (use case only) | `false` *(inherited)* | Only useful for single-wall hollow vase models. Enable per-model when actually needed. **DECISION: no change** — leave off in base profile. |
| Smooth spiral | `false` | `false` *(default)* | (default) | Only relevant if spiral vase is on. **DECISION: no change** — default off. |
| Timelapse type | `traditional` | `traditional` *(default)* | (default) | `traditional` works for any printer. **DECISION: no change** — default is correct. |
| Fuzzy skin | `none` | `none` (decorative use only) [^fuzzy-skin] | (default `none`) *(inherited)* | Decorative textured finish. Enable per-model when wanted. **DECISION: no change** — leave off in base profile. |
| Fuzzy skin thickness | 0.3 mm | 0.1-0.4 mm (when used) [^fuzzy-skin] | (default 0.3) | Only matters if fuzzy skin is on. **DECISION: no change** — default 0.3 is the community sweet spot. |
| Fuzzy skin point distance | 0.8 mm | 0.1-0.4 mm for fine texture, 0.8 mm default [^fuzzy-skin] | (default 0.8) | Only matters if fuzzy skin is on. **DECISION: no change** — moot. |
| Fuzzy skin first layer | `false` | `false` *(default)* | (default) | Off = don't fuzzify first layer (better adhesion). **DECISION: no change** — default off is correct. |

### G-code output

| Setting | OrcaSlicer default | Researched value | Our value | Comments |
|---|---|---|---|---|
| Filename format | `{input_filename_base}_{filament_type[initial_tool]}_{print_time}.gcode` | (varies by user) | (inherited) | Templated output filename. Default includes object name, filament, print time. **DECISION: no change** — default is sensible. |
| Verbose G-code | `false` | `false` *(default for performance)* | (default) | Adds extensive comments to gcode. Useful for debugging slicer bugs. **DECISION: no change** — off in production for smaller files. |
| Label objects | `false` | `false` *(default)* | (default) | Adds object labels so Klipper's `exclude_object` (already enabled in our Klipper config) can cancel individual objects mid-print. Without this, the feature has nothing to exclude. **DECISION: CHANGE to `true`** — activates our Klipper exclude_object capability. |
| Exclude object | `1` (Klipper common) | `1` [^vzbot-bundled] | `1` *(inherited)* | Emits EXCLUDE_OBJECT gcode commands. **DECISION: no change** — default on is correct (pairs with the label_objects change above). |
| G-code comments | `false` | `false` *(default)* | (default) | Lighter-weight than verbose gcode. **DECISION: no change** — off for production. |
| Reduce infill retraction | `1` | `1` *(default)* | (default) | Suppresses retractions during infill. **DECISION: no change** — default on is correct. |

### Post-processing scripts

| Setting | OrcaSlicer default | Researched value | Our value | Comments |
|---|---|---|---|---|
| Post-processing scripts | (empty) | **`/Users/user/klipper_estimator --config_file /Users/user/klipper_estimator.cfg post-process`** | (empty — needs setting) | Klipper Estimator is installed. After setting, slicer gets accurate time estimates instead of the optimistic guess. **DECISION: ADD the klipper_estimator post-process command**. Path: `/Users/user/klipper_estimator --config_file /Users/user/klipper_estimator.cfg post-process`. |

### Notes

| Setting | OrcaSlicer default | Researched value | Our value | Comments |
|---|---|---|---|---|
| Notes | (empty) | (varies) | (empty) | Free-form notes on the process profile. We have this markdown file for that purpose. **DECISION: no change** — leave empty. |

---

## Summary of recommended Others tab changes

The Others tab is **mostly use-case dependent** rather than quality-dependent. Most settings should stay at defaults unless you have a specific reason.

### Recommended changes (small QoL improvements)

| Setting | Currently | Recommend | Reason |
|---|---|---|---|
| `post_process` | (empty) | `/Users/user/klipper_estimator --config_file /Users/user/klipper_estimator.cfg post-process` | **Manual action** — paste into Orca UI. Activates Klipper Estimator for accurate print time estimates. Set up earlier this session. |
| `label_objects` | `false` | `true` | Enables object labels in gcode so Klipper's `exclude_object` feature actually works (allows cancelling individual objects mid-print). Klipper common already enables `exclude_object: 1`, but without `label_objects: true` in the slicer there's nothing to exclude. |

### Optional (enable if you want)

| Setting | Currently | Could enable | Reason |
|---|---|---|---|
| `skirt_loops` | 0 | 1 | Visual sanity check that the bed is clean and Z is correct before each print. The Beacon makes this technically redundant, but a 1-loop skirt costs ~10 seconds and gives you a "yes, the bed is good" confirmation by eye. |

### Confirmed correct (no change needed)

- `brim_type: no_brim` — correct for PLA on textured PEI
- `print_sequence: by layer` — standard mode, supports filament change at layer
- `spiral_mode: false` — correct for general printing (enable per-model when actually printing a vase)
- `fuzzy_skin: none` — correct for general quality printing (enable per-model when you want texture)
- `exclude_object: 1` — Klipper-friendly, default in Klipper common
- `reduce_infill_retraction: 1` — quality and speed benefit
- All other defaults — use-case dependent, no specific reason to change

### Settings that aren't really "quality" decisions

Most of the Others tab is one of these three categories:

1. **Use-case toggles** (spiral mode, fuzzy skin, by-object) — enable per-model when you specifically want that mode, leave off in the base profile
2. **Output formatting** (filename, verbose, comments) — personal preference, no quality impact
3. **Hardware-specific** (timelapse type, exclude_object) — already correct for our setup

There's no "wrong" setting on the Others tab, just inappropriate ones for specific use cases.

---

## References

[^skirt-brim]: OrcaSlicer Skirt and Brim Wiki + community guidance. Skirt: 1-3 loops at 5 mm distance is standard, but skirt does **not** improve adhesion — its purpose is nozzle priming and visual first-layer verification. Brim: 5-8 mm width with 0-0.12 mm object gap when needed. **Modern textured PEI plates have largely eliminated the need for brims on PLA**, so `no_brim` is the correct default for our setup. <https://github.com/SoftFever/OrcaSlicer/wiki/others_settings_skirt> · <https://github.com/SoftFever/OrcaSlicer/wiki/others_settings_brim> · <https://help.prusa3d.com/article/skirt-and-brim_133969>

[^print-sequence]: OrcaSlicer print sequence — `by layer` is the default and standard mode (prints all objects layer-by-layer simultaneously). `by object` prints each object completely before moving to the next, useful only when objects need to be physically separated post-print or when failure isolation matters. `by layer` supports filament change / pause at layer; `by object` does not. For general PLA quality printing, `by layer` is correct. <https://github.com/SoftFever/OrcaSlicer/discussions/4870>

[^fuzzy-skin]: OrcaSlicer Fuzzy Skin Wiki and community guidance. Default thickness 0.3 mm (range 0.1-2.0 mm, practical range 70-125 % of nozzle diameter). Default point distance 0.8 mm. Community sweet spot for fine even texture: 0.10 mm thickness and 0.10 mm point distance. **Use cases**: decorative grip surfaces, hiding layer lines, organic finishes. **Not useful for general PLA quality printing** — fuzzy skin is the opposite of "smooth quality finish". Enable per-model when actually wanted. <https://github.com/SoftFever/OrcaSlicer/wiki/others_settings_fuzzy_skin> · <https://help.prusa3d.com/article/fuzzy-skin_246186> · <https://wiki.bambulab.com/en/software/bambu-studio/parameter/fuzzy-skin>

[^vzbot-bundled]: See the other tab files for the full reference. The VzBot bundled profile sets `exclude_object: 1` in its common base, matching the Klipper common base. <https://github.com/SoftFever/OrcaSlicer/tree/main/resources/profiles/Vzbot/process>
