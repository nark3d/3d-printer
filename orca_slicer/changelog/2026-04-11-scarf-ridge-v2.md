# Scarf ridge fix v2 — restart-extra fine-tune — 2026-04-11

> Follow-up to `2026-04-11-scarf-ridge-fix.md`. The 10%→15% clearance bump didn't fully kill the ridge. Pivoting from "give the blob more room" to "make the blob smaller", and reverting the 15% change so we isolate the variable cleanly.

## Starting state

After applying `2026-04-11-scarf-ridge-fix.md` (`seam_slope_start_height: 10% → 15%`), the user reported:

- Normal seam: still "pretty much perfect", happy with it
- Scarf entry: **still has a small step up** on the outside
- Confirmation that the clearance approach alone isn't enough

### Bonus data from accidental test

User accidentally sliced with `seam_slope_type: Contour and hole` (= `"all"` in JSON) instead of Contour only for this test. Result: the back seam was also a scarf, and it showed the same ridge at its entry point. This is a useful empirical confirmation — the ridge is **per-feature** (every scarf start), not location-specific. Supports the "blob at de-retract" theory over any "specific geometry" alternative.

## What we learned from v1

Raising `seam_slope_start_height` from 10% to 15% gave the blob more room to dissipate — 0.02 mm → 0.03 mm of Z clearance. The ridge got smaller but didn't disappear. This tells us:

1. The blob is still too large at 0.05 mm restart_extra — even with 50% more clearance, it's piling up
2. Clearance alone can't fix this without going to comically high start heights (20%+ would start to compromise the scarf's anti-visibility effect)
3. The real fix is to reduce the blob itself

## Fix

Two changes, applied together:

### Change 1 — `seam_slope_start_height`: 15% → 10% (revert)

**File**: `process/0.20mm Standard @MyKlipper - PLA.json`

**Why revert**: isolating the variable. Since 15% didn't fully fix the ridge, leaving it at 15% would stack two partial changes and muddy the analysis of v2. Reverting to 10% means the next test isolates exactly one change — the restart_extra reduction — so we can attribute any improvement cleanly. Also leaves scarf geometry at its theoretically-best value (low start height = more scarf blending).

### Change 2 — `filament_retract_restart_extra`: 0.05 → 0.04

**File**: `filament/base/Sunlu PLA Matte @MyKlipper 0.4 nozzle.json`

**What it does**: reduces the bonus-material-after-de-retract from 0.05 mm to 0.04 mm per de-retract event.

**Why this should work**: the ridge is formed from the 0.05 mm blob being squeezed sideways at scarf start. Shrinking the blob by 20% should shrink the ridge by ~20%. If the ridge at 0.05 is "very, very small" (user's words), 20% smaller should put it below perceptibility.

**Normal seam risk assessment**:

Tracking the history of this value for this filament:
| Value | Test result |
|---|---|
| 0.03 (original) | Visible dip + feelable step at traditional seam start |
| 0.10 (round 1) | Normal seam great, but huge scarf bulge — untenable |
| 0.05 (round 2) | Normal seam "pretty much perfect", scarf ridge "very small" |
| **0.04 (v2)** | Normal seam likely nearly perfect (interpolates closer to 0.05 than 0.03), scarf should improve |

0.04 sits 80% of the way from the "bad" end (0.03) to the "pretty much perfect" end (0.05), much closer to the perfect end. The normal seam should remain good — any regression will be minor and pixel-peeping territory.

## Fallback plan

If the ridge still shows at v2:

1. **Try 0.035** — next small step down. Still above the 0.03 "bad" threshold but closer to it
2. **Go 0.03 + 15% start height** — combine reverting to original blob size with more clearance. Compromises normal seam quality slightly but kills the ridge
3. **Accept the ridge as below "good enough" threshold** — "very, very small" at 0.05 was already nearly invisible

## Note on file state

When I attempted to edit the filament file, it was already at `0.04` — user appears to have made this change directly via Orca UI or file edit before I got to it. Re-read confirmed the target state. The process file was edited by me via the Edit tool (15% → 10%).

## Backup

Full pre-change snapshot of all three profiles saved at:
`orca_slicer/backups/2026-04-11-scarf-ridge-v2/`

## Applied changes summary

| # | File | Key | Before | After |
|---|---|---|---|---|
| 1 | `process/0.20mm Standard @MyKlipper - PLA.json` | `seam_slope_start_height` | `15%` | `10%` |
| 2 | `filament/base/Sunlu PLA Matte @MyKlipper 0.4 nozzle.json` | `filament_retract_restart_extra` | `0.05` | `0.04` |
