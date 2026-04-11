# OrcaSlicer Printer Settings — Motion Ability tab

> One file per tab. See [`README.md`](README.md) for the index of all tab documents and shared hardware context.
>
> **This is the highest-impact tab in the printer settings folder.** The Motion Ability tab controls the slicer's understanding of the printer's velocity, acceleration, and jerk limits — and uses those values for *gcode planning*. If they're wrong, the slicer plans suboptimal gcode based on the wrong reality, even though Klipper itself enforces the correct limits at execution time.

---

## ⚠️ How OrcaSlicer interacts with Klipper on motion limits

This is critical context for everything below. **Klipper and OrcaSlicer treat motion limits differently** and the interaction has consequences.

### The two-layer reality

| Layer | Where the values live | What they're used for |
|---|---|---|
| **Klipper** (`printer.cfg`) | `[printer] max_velocity`, `max_accel`, `max_z_velocity`, `max_z_accel`, `square_corner_velocity` | **Hard enforcement at execution time.** Klipper *cannot* exceed these values regardless of what gcode it receives |
| **OrcaSlicer** (this tab) | `machine_max_speed_*`, `machine_max_acceleration_*`, `machine_max_jerk_*` | **Slicer-side gcode planning.** Orca uses these for time estimates, velocity ramping, deceleration planning around corners |

**Klipper ignores M201/M203/M204/M205 commands from gcode by default** — it parses them but does not let them override the `printer.cfg` ceiling. So if the slicer emits `M201 X40000 Y40000` (max accel 40 000), Klipper reads it but still enforces 10 000 from `printer.cfg`.

### What goes wrong if the slicer values are higher than Klipper's

The slicer plans gcode assuming it has accel headroom that doesn't actually exist:

1. **Wrong time estimates** — slicer thinks "40 mm at 600 mm/s with 40 000 accel = X seconds" but Klipper actually does it in Y seconds (slower because actual accel is 4× lower)
2. **Wrong velocity ramping at corners** — slicer thinks the printer can decelerate from 600 mm/s to 100 mm/s in a tight feature with 40 000 accel = ~12 mm of deceleration distance. With actual 10 000 accel, the printer needs ~50 mm. If the feature is shorter than 50 mm, Klipper has to slow down further than the slicer planned for, producing visible velocity discontinuities
3. **Suboptimal feature stitching** — features that could be combined into smoother motion get split because the slicer's planning estimates don't match reality

### What goes wrong if the slicer values are lower than Klipper's

Mostly nothing — the slicer is being conservative, gcode is just slightly slower than necessary. **It's safer to be too low than too high** in the slicer.

### The right answer

**Set the slicer's machine_max_* values to *exactly match* the Klipper `printer.cfg` values.** This makes the slicer's planning honest. Klipper Estimator (which we have installed) post-processes the gcode and overrides print time estimates anyway, so even if slicer time estimates are slightly off, the final estimate is accurate.

---

## Klipper printer.cfg motion limits (the source of truth)

Pulled directly from `~/printer_data/config/printer.cfg`:

```
max_velocity: 600
max_accel: 10000
max_z_velocity: 5
max_z_accel: 50
```

Plus implicit:
- `square_corner_velocity`: not explicitly set in printer.cfg, so Klipper uses its **default of 5.0 mm/s**
- `minimum_cruise_ratio`: not explicitly set, so Klipper uses its default 0.5

These are **the values OrcaSlicer should match** for honest gcode planning.

---

## ⚠️ Findings flagged at the top

| Setting | OrcaSlicer has | Klipper actually does | Mismatch |
|---|---|---|---|
| `machine_max_acceleration_x` (normal) | **40 000 mm/s²** | 10 000 mm/s² | **4× too high** |
| `machine_max_acceleration_y` (normal) | **40 000 mm/s²** | 10 000 mm/s² | **4× too high** |
| `machine_max_acceleration_extruding` (normal) | **40 000 mm/s²** | 10 000 mm/s² (uses same `max_accel` for all moves) | **4× too high** |
| `machine_max_acceleration_retracting` (normal) | **40 000 mm/s²** | 10 000 mm/s² | **4× too high** |
| `machine_max_acceleration_travel` (normal) | (inherited 20 000) | 10 000 mm/s² | **2× too high** |
| `machine_max_speed_z` (normal) | **20 mm/s** | 5 mm/s | **4× too high** |
| `machine_max_acceleration_z` (normal) | (inherited 500) | 50 mm/s² | **10× too high** |
| `machine_max_speed_x/y` (normal) | 600 mm/s | 600 mm/s | ✅ correct |

**The Z-axis values are particularly mismatched.** The Beacon-equipped printer with three Z motors and Oldham couplers has been tuned for 5 mm/s Z velocity and 50 mm/s² Z accel — those are real, deliberate values reflecting the mechanical limits of the Z drive train. The slicer thinks Z can do 4× the velocity and 10× the accel — that affects Z hop planning, layer change timing, and bed mesh probe move timing.

---

## Maximum speed

OrcaSlicer's speed values are pairs: `[normal, silent]` (newer Orca) or just `[normal]` (older). The "silent" mode is for printers with a quiet operating profile — **not applicable to Klipper** since Klipper doesn't have a stealth mode toggle. The first value (normal) is what matters; the second is irrelevant for our use case.

| Setting | OrcaSlicer default (Klipper base) | Researched value | Our value | Comments |
|---|---|---|---|---|
| **Max X velocity** | 500, 200 | **600** [^klipper-printer-cfg] | **600, 200** | Matches Klipper `max_velocity: 600`. **DECISION: no change** — override is correct. |
| **Max Y velocity** | 500, 200 | **600** [^klipper-printer-cfg] | **600, 200** | Same. **DECISION: no change** — override is correct. |
| **Max Z velocity** | 12, 12 | **5** [^klipper-printer-cfg] | **20, 12** | **MISMATCH — Orca thinks Z can do 20 mm/s but Klipper enforces 5 mm/s.** Klipper's `max_z_velocity: 5` is deliberate for our 3-Z Beacon-equipped drivetrain. **DECISION: CHANGE to `5, 5`** to match Klipper exactly. Makes Z hop and layer change timing honest in slicer planning. |
| **Max E velocity** | 25, 25 | **120** for our Orbiter 2.5 | **120, 25** | Klipper's `[extruder] max_extrude_only_velocity` is what actually constrains E — 120 here is slicer-planning only. Orbiter 2.5 has very low backlash and handles aggressive E moves. **DECISION: no change** — override defensible, supports the calibrated 120 mm/s retraction speed. |

---

## Maximum acceleration

The most consequential section. **All five XY/extruder accel settings should be brought down to match Klipper's `max_accel: 10 000`.**

| Setting | OrcaSlicer default (Klipper base) | Researched value | Our value | Comments |
|---|---|---|---|---|
| **Default (max accel)** | 20 000, 20 000 | (not directly set in this tab — `max_accel` is in printer.cfg only) | — | OrcaSlicer has no "default acceleration" on this tab — there are per-feature accel rows below instead. **DECISION: N/A** — no direct setting exists. |
| **Max X acceleration** | 20 000, 20 000 | **10 000** [^klipper-printer-cfg] | **40 000, 20 000** | **4× too high.** Klipper's `max_accel: 10 000` is the hard ceiling. **DECISION: CHANGE to `10 000, 5 000`**. Highest-impact setting on this tab — makes slicer velocity planning honest. |
| **Max Y acceleration** | 20 000, 20 000 | **10 000** [^klipper-printer-cfg] | **40 000, 20 000** | Same as X. **DECISION: CHANGE to `10 000, 5 000`**. |
| **Max Z acceleration** | 500, 200 | **50** [^klipper-printer-cfg] | (inherited 500, 200) | **10× too high.** Klipper `max_z_accel: 50` is the mechanical limit of the 3-Z drivetrain. **DECISION: CHANGE to `50, 50`**. Makes Z planning match reality. |
| **Max E acceleration** | 5 000, 5 000 | (constrained by Klipper `max_extrude_only_accel`, not `max_accel`) | **3 000, 5 000** | Extruder accel. Lower than default — user explicitly reduced this. Defensible if it was tuned to prevent Orbiter skipping. Klipper enforces its own `max_extrude_only_accel` separately. **DECISION: no change** — empirically tuned, leave alone. |
| **Max acceleration when extruding** | 20 000, 20 000 | **10 000** [^klipper-printer-cfg] | **40 000, 20 000** | Slicer's accel ceiling for **printing moves**. **4× too high.** **DECISION: CHANGE to `10 000, 5 000`**. |
| **Max acceleration when retracting** | 5 000, 5 000 | **10 000** [^klipper-printer-cfg] | **40 000, 5 000** | Used during retract+move sequences. Klipper uses `max_accel` for this too. **8× too high on normal.** **DECISION: CHANGE to `10 000, 5 000`**. |
| **Max travel acceleration** | 20 000, 20 000 (inherited) | **10 000** [^klipper-printer-cfg] | (inherited 20 000) | Slicer's accel ceiling for non-extruding travel. Klipper uses same `max_accel` for travel and printing. **2× too high.** **DECISION: CHANGE to `10 000, 5 000`** (explicit override instead of inherited). |

### Why these matter even though Klipper enforces the actual limits

The slicer doesn't *just* emit motion commands — it **plans the velocity profile** of every move based on these accel values. With wrong values:

- **Inner-feature deceleration planning** is wrong → slicer assumes it can decelerate faster than reality → tight features get gcode that requires impossible velocity drops
- **Print time estimates** are wrong → optimistic by 1.5-3× compared to reality
- **Pressure advance compensation timing** is wrong → the slicer's PA-compensation calculations assume the wrong accel curve
- **Junction deviation / corner velocity** decisions are wrong → corners that the slicer thinks the printer can take cleanly actually need more deceleration

Klipper Estimator (which we have installed) post-processes the gcode and corrects the *time estimate*, but it can't fix the velocity planning that was already encoded in the gcode based on wrong assumptions.

---

## Maximum jerk

Klipper does **not** use jerk in the Marlin sense. Klipper uses `square_corner_velocity` (default 5.0 mm/s) which is a different mathematical model — the maximum velocity at which the printer can take a 90° corner without changing speed.

OrcaSlicer's jerk settings are **slicer-side planning only** for Klipper users. Klipper ignores any M205 commands the slicer emits. The values here should be set to what the slicer would *plan as if* the printer had that jerk — for Klipper with `square_corner_velocity: 5.0`, the equivalent jerk is in the **5-9 mm/s range**.

| Setting | OrcaSlicer default (Klipper base) | Researched value | Our value | Comments |
|---|---|---|---|---|
| **Max X jerk** | 9, 9 | **~7** (mathematically equivalent to SCV 5.0: jerk = SCV × √2 ≈ 7.07) or 5-9 range | (inherited 9, 9) | Slicer-side jerk is **planning only** — Klipper ignores M205. The 9 default is in the range and matches the Klipper-common profile. Changing to 7 would be mathematically more accurate but has no practical effect because Klipper Estimator fixes time estimates anyway. **DECISION: no change** — keep inherited 9. |
| **Max Y jerk** | 9, 9 | **5-9** | (inherited 9, 9) | Same reasoning as X. **DECISION: no change**. |
| **Max Z jerk** | 0.2, 0.4 | **0.4** | (inherited 0.2, 0.4) | Z jerk for layer changes. Klipper-base default is fine; Klipper handles Z layer-change deceleration internally regardless. **DECISION: no change**. |
| **Max E jerk** | 2.5, 2.5 | **2.5-5** | **5, 2.5** | Override: 5 normal / 2.5 silent. Higher E jerk allows faster retraction transitions. Orbiter 2.5's low backlash handles aggressive E moves. **DECISION: no change** — override defensible for the Orbiter. |

---

## Other motion settings (not visible in the screenshot)

Some Motion Ability settings may exist in newer OrcaSlicer versions but not show up in the screenshot:

| Setting | What it does | Recommendation for Klipper | Decision |
|---|---|---|---|
| Min extruding rate | Minimum velocity the slicer will plan for printing moves | Default 0 is fine — this is a floor below which the slicer won't plan, and 0 means "no floor, let per-feature speeds decide" | **DECISION: no change** — leave at default 0 |
| Min travel rate | Minimum velocity the slicer will plan for travel moves | Same — default 0 is correct | **DECISION: no change** — leave at default 0 |

---

## Summary of recommended Motion Ability tab changes

### Strongly recommended (fixes Klipper mismatches)

| Setting | Currently | Recommend | Reason |
|---|---|---|---|
| `machine_max_acceleration_x` | **40 000, 20 000** | **10 000, 5 000** | Match Klipper `max_accel: 10 000`. Slicer planning will be honest |
| `machine_max_acceleration_y` | **40 000, 20 000** | **10 000, 5 000** | Same |
| `machine_max_acceleration_extruding` | **40 000, 20 000** | **10 000, 5 000** | Same |
| `machine_max_acceleration_retracting` | **40 000, 5 000** | **10 000, 5 000** | Same |
| `machine_max_acceleration_travel` | (inherited 20 000) | **10 000, 5 000** | Same |
| `machine_max_speed_z` | **20, 12** | **5, 5** | Match Klipper `max_z_velocity: 5`. Z hop and layer change timing planning will be accurate |
| `machine_max_acceleration_z` | (inherited 500, 200) | **50, 50** | Match Klipper `max_z_accel: 50`. Z move planning accurate |

### Optional refinements

| Setting | Currently | Could refine to | Reason |
|---|---|---|---|
| `machine_max_jerk_x/y` | (inherited 9) | 5 | Match Klipper `square_corner_velocity: 5.0` more closely. Both work; 5 is more accurate |

### Confirmed correct (no change needed)

- `machine_max_speed_x`: 600 — matches Klipper `max_velocity`
- `machine_max_speed_y`: 600 — same
- `machine_max_speed_e`: 120 — Orbiter 2.5 can handle this for retraction; Klipper enforces `max_extrude_only_velocity` separately
- `machine_max_acceleration_e`: 3 000 — defensible if tuned to prevent extruder skipping
- `machine_max_jerk_e`: 5 — defensible for Orbiter's low backlash

---

## What changes in print behaviour after fixing the mismatches?

**Nothing visible immediately.** Klipper was already enforcing the real limits — print quality was already what the actual hardware produces. Fixing the slicer-side values doesn't change *what the printer does*, it changes *what the slicer plans for*.

What does change:

1. **Slicer time estimates become more accurate** — closer to actual print time even before Klipper Estimator post-processes
2. **Tight-feature deceleration planning becomes honest** — the slicer no longer plans corners assuming 4× the actual accel headroom
3. **Pressure advance compensation timing becomes accurate** — the slicer's per-feature PA timing assumptions match reality
4. **The gcode is "honest"** — what's in the file matches what Klipper will do

In practice the effects are subtle. **The biggest user-visible benefit is more accurate time estimates** (especially before Klipper Estimator post-processes) and better corner-feature handling on small features that hit the deceleration ceiling.

---

## Why did the profile have the wrong values in the first place?

Likely history (speculation):

1. The user originally tuned a more aggressive Klipper config with `max_accel: 40 000` (which matches the OrcaSlicer values)
2. Later, after input shaper tuning and discovering ringing at high accel, Klipper's `max_accel` was reduced to 10 000 — but the OrcaSlicer profile wasn't updated
3. The mismatch has been quietly there ever since

This is **a maintenance hazard** — any time you change `printer.cfg` motion limits, you have to remember to update OrcaSlicer to match. There's no automatic sync. The fact that we're catching this now is valuable.

---

## References

[^klipper-printer-cfg]: Klipper `printer.cfg` motion limits — the source of truth for what the printer can actually do. Klipper enforces `max_velocity`, `max_accel`, `max_z_velocity`, `max_z_accel`, and `square_corner_velocity` as hard ceilings at execution time. Slicer commands (M201/M203/M204/M205) are parsed but ignored — Klipper does *not* let the slicer override these. Source: `~/printer_data/config/printer.cfg` on the printer. Cross-reference: [`HARDWARE.md`](../../HARDWARE.md) line 235-236.
