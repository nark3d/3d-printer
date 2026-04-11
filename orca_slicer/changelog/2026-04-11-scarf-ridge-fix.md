# Scarf ridge fix — 2026-04-11

> Single-setting surgical tweak to eliminate the residual scarf-entry ridge observed after the round-2 test print, without disturbing the perfect-now traditional seam.

## Starting state

After the round-2 tuning pass (`orca_slicer/changelog/2026-04-11-test-print-tune.md`), the user reported the following from test print evaluation:

- **Ironing (concentric pattern)**: massive win — drag marks eliminated, surface clean
- **Traditional seam** (`seam_slope_type: external` on hole interior): "perfect, pretty much"
- **Scarf seam** (on outer wall): "very, very small ridge" still visible at scarf entry

The scarf bulge from the round-1 `filament_retract_restart_extra: 0.10` regression is gone at the round-2 value of `0.05`, but a small residual ridge remains.

## Root cause

The `filament_retract_restart_extra: 0.05` setting deposits a 0.05 mm blob of filament every time the extruder de-retracts. This is deliberate — it compensates for the micro-gap / dip at traditional seam starts caused by pressure advance. The user has confirmed that 0.05 is exactly right for traditional seams.

The problem is that `filament_retract_restart_extra` is a **global** filament-level setting — it applies to every de-retract, including the one right before a scarf joint starts. Unlike a traditional seam (where the nozzle is at full Z and full flow, so a tiny bonus blob fuses invisibly into the wall), a scarf joint begins at:

- **Z offset**: `seam_slope_start_height: 10%` of layer height = 0.02 mm above the previous layer (with 0.20 mm layers)
- **Flow**: 0% of nominal, ramping up over `seam_slope_steps: 20` steps across ~20 mm

At 0.02 mm nozzle clearance, the 0.05 mm restart blob has nowhere to go — it can't flow upward into the scarf surface (no space) or downward (already touching the previous layer), so it squeezes sideways. That sideways squeeze is the ridge.

**We can't lower `filament_retract_restart_extra` without hurting the now-perfect traditional seam.** Any fix must be targeted at the scarf start specifically.

## Fix

### Change 1 of 1 — `seam_slope_start_height`: 10% → 15%

**File**: `process/0.20mm Standard @MyKlipper - PLA.json`

**What it does**: raises the initial nozzle Z at the start of a scarf ramp from 10% of layer height to 15% of layer height.

**With 0.20 mm layers**:
- Before: scarf starts at 0.02 mm above previous layer (10% × 0.20)
- After: scarf starts at 0.03 mm above previous layer (15% × 0.20)

**Why this works**: 50% more vertical clearance for the 0.05 mm restart blob to dissipate into the scarf ramp surface rather than pile up as a sideways squeeze. 0.03 mm is still well below the layer height, so the scarf ramp continues to do its job of hiding the seam — we're not abandoning the scarf, just giving it slightly more breathing room at the very start.

**Why not other fixes**:
- Lowering `filament_retract_restart_extra` — would regress the now-perfect traditional seam
- Lowering `retraction_length` — global effect, affects every retraction everywhere
- Lowering `scarf_joint_speed` — slower start could help pressure equalise, but 75% is already conservative and the ridge mechanism is volumetric not dynamic
- Disabling retraction before scarf — OrcaSlicer has no such toggle; the retract happens during layer-change travel

Incrementing just the scarf start height is the most surgical change available — it touches exactly one variable and affects only the scarf ramp entry.

## Fallback plan (if this doesn't fully kill the ridge)

Documented here so next-session-me doesn't have to rediscover them:

1. **Try 20%** (0.04 mm clearance) — doubles the headroom, still invisible on 0.20 mm layers
2. **Lower `scarf_joint_speed` 75% → 50%** — more time for pressure to equalise before flow ramps up
3. **Lower `filament_retract_restart_extra` 0.05 → 0.04** — last resort, risks tiny traditional-seam dip regression

## Backup

Full pre-change snapshot of all three profiles saved at:
`orca_slicer/backups/2026-04-11-scarf-ridge-fix/`

## Applied changes summary

| # | File | Key | Before | After |
|---|---|---|---|---|
| 1 | `process/0.20mm Standard @MyKlipper - PLA.json` | `seam_slope_start_height` | `10%` | `15%` |
