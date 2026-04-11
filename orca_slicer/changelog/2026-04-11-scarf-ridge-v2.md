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

## Test result — 2026-04-11 (partial success, paused here)

User ran the scarf v2 test print and reported:

- **Scarf seam**: still has the visible step bands we talked about (the ~20 slope-step artefacts that round 1's ERS 300→150 reduction was meant to soften). Improved vs earlier rounds but not eliminated.
- **Traditional seam**: "a little pronounced" at the v2 value of `filament_retract_restart_extra: 0.04`. This is a slight regression from the 0.05 state which the user previously called "pretty much perfect". Predictable outcome — at 80% of the way from "bad" (0.03) to "perfect" (0.05), the normal seam was always going to lose a bit of quality.
- **Scarf ridge (the v2 target)**: not explicitly mentioned — outcome ambiguous. The "still has the steps" could be referring either to the slope_steps banding or to residual entry ridge.
- **User verdict**: "happy enough for now, will have another go at it another day"

### Outcome: partial win, session paused

- Ironing: fully solved ✅ (concentric pattern eliminated drag marks)
- Seams: improved vs session start but not resolved. **Paused at an acceptable-not-perfect state** and accepted as good enough for now.

### Candidate next levers (when user picks this back up)

Documented here so next-session-me doesn't start from zero:

1. **Reduce `seam_slope_steps` from 20 → 10** — fewer, wider slope segments. Theory: makes each transition more blended and reduces the count of visible bands. Orthogonal to everything we've tried so far.
2. **Re-calibrate `pressure_advance`** — currently `0.045`, calibrated before the 2026-04-10 shaper retune. Potentially stale. If PA is slightly wrong, every seam start would show a small artefact from pressure mismatch, which could explain both the scarf banding AND the normal seam visibility.
3. **Go back to `filament_retract_restart_extra: 0.05`** — restores the "pretty much perfect" normal seam, accepting that the scarf ridge is slightly bigger. Trade the normal seam loss for the scarf gain if that's the wrong side of the Pareto front.
4. **Try `seam_slope_type: all` with `seam_slope_conditional: 1` and tighter thresholds** — makes scarf applicable only to flatter/shallower features, reducing the number of scarf entries per print (fewer chances to show visible entry artefacts on features where a normal seam would do just as well).
5. **Disable `staggered_inner_seams` temporarily** — it's currently on. Theory: the inner-wall seam stagger might be amplifying visible seam artefacts at the outer wall by creating inconsistent bond timing at seam positions. Unverified.
6. **Investigate `enable_pressure_advance` / flow ratio combined effects** — current `filament_flow_ratio: 0.98` and `pressure_advance: 0.045`. If flow is fractionally off and PA is fractionally off, the two errors can stack at seam starts.

None of these is obviously "the answer" — this needs another round of empirical testing when the user has time.
