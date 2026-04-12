# Next steps — resume point after 2026-04-11 slicer tuning session

> This file is the resume point for whoever picks up the project next (you, or future-Claude). Read this first. It supersedes the pre-session version of this doc.
>
> **Current position**: Mechanical tuning (Phases 1-3, 2026-04-10) and OrcaSlicer tuning (Phase 6, 2026-04-11) are both at a "good enough for production" state. Ironing is fully solved. Seams are improved but paused at an acceptable-not-perfect state with documented residual issues — the user explicitly chose to stop and revisit another day. **No blocking next action** — the printer is ready for real print work.

---

## Status as of end-of-session 2026-04-10

For full session detail with all measurement data, graphs, and the lessons learned, see **[`tuning/2026-04-10/README.md`](tuning/2026-04-10/README.md)** (7 sections, 14 reference PNGs, ~500 lines).

### Phase status

| Phase | Description | Status |
|---|---|---|
| 1.1 | `AXES_MAP_CALIBRATION` | ✅ Done — `accel_axes_map: -x, -y, z` applied to Beacon |
| 1.2 | `COMPARE_BELTS_RESPONSES` | ✅ Done — 96.4% similarity ("excellent") on Beacon, ~87% on LIS2DW (the LIS2DW number is the honest baseline) |
| 1.3 | `AXES_SHAPER_CALIBRATION` | ✅ Done — final saved values: MZV X=61.0 Hz ζ=0.059, MZV Y=42.0 Hz ζ=0.076 (LIS2DW-derived) |
| 2 | Decision point | ✅ "All green" — gantry mechanically healthy, no remediation needed |
| 3.1 | `CREATE_VIBRATIONS_PROFILE` | ✅ Done — 71.4% polar symmetry initially, recovered to 82.1% after the autotune detour. Motor main resonance 166.4 Hz, ζ=0.083 |
| 3.2 | Chamber heater PID re-cal | ⛔ **Skipped by Adam's decision** — see `memory/chamber_pid_skip.md`. Don't propose unprompted. Current values (Kp 100 / Ki 0.3 / Kd 30) work well enough; can be hand-tuned if behaviour is wrong |
| 3.3 | TMC Autotune enablement | ⛔ **Tried, regressed, reverted** — see `tuning/2026-04-10/README.md` §7. Made polar symmetry worse (71→51%), the official Klipper docs say *"For most users, no special TMC tuning is required."* See `memory/dont_push_unneeded_tuning.md`. **Do not recommend on this printer** unless concrete print-time symptoms appear |
| 4 | Per-filament Orca calibration | 🔵 Done for SUNLU PLA Matte. Re-run per new spool, not per profile |
| 5.1 | KAMP adaptive meshing | 🔵 Already have Klipper's built-in `BED_MESH_CALIBRATE ADAPTIVE=1` in PRINT_START — KAMP's own adaptive module is essentially the same. **KAMP's other modules** (Smart Park, Line Purge, Voron Purge) are not enabled and may be worth doing — pending Adam's decision |
| 5.2 | KAMP Smart Park | ⛔ Pending decision (worth enabling — prevents nozzle ooze landing on print start area) |
| 5.3 | Klipper Estimator install | ✅ Done — binary on Mac at `/Users/user/klipper_estimator`, also on printer at `~/klipper_estimator`. Orca post-processing line is in the next section |
| 5.4 | Beacon Contact mode + thermal expansion compensation | ⏳ Deferred — bigger change worth its own session if first-layer squish varies between prints |
| 6 | OrcaSlicer seam + ironing tune (2026-04-11) | 🟡 **Partial — paused at acceptable state** — ironing fully solved (concentric pattern, 12% flow, 40 mm/s — no drag marks). Seams improved but not perfect: scarf still shows visible step bands, normal seam slightly pronounced at `restart_extra: 0.04`. User accepted "good enough for now, will revisit another day". Full trail in `orca_slicer/changelog/2026-04-11-*.md` and per-round backups at `orca_slicer/backups/2026-04-11-*/` |

### Cold-state hardware health (measured 2026-04-10)

| Check | Result | Verdict |
|---|---|---|
| Beacon Z probe σ | **60 nm** range 335 nm at 10 samples | Top decile |
| Z-tilt convergence | 11.7 µm initial → 0.77 µm in 1 retry | Effectively flat |
| Belt tension (physical) | Both belts at **111 Hz** measured by Adam | Matched |
| Polar vibration symmetry | 82.1 % (post-autotune-revert) | Typical for heavy CNC toolhead |
| Belt similarity (LIS2DW) | ~87 % reproducible to <1 % | "Good" band |
| Motor resonance | 166.4 Hz, ζ=0.083 | Healthy |

### Currently saved tuning values (post-2026-04-10 session)

| Setting | Value | Notes |
|---|---|---|
| `max_velocity` | 600 mm/s | Unchanged |
| `max_accel` | 10 000 mm/s² | Above the EI shaper ceiling of ~5400, but the Y is now MZV which has different smoothing characteristics. Worth revisiting after observing real print quality |
| **Input shaper X** | **MZV @ 61.0 Hz, ζ 0.059** | Was 63.8 Hz before this session — drift over time |
| **Input shaper Y** | **MZV @ 42.0 Hz, ζ 0.076** | Was 43.4 Hz; type was MZV, briefly tried EI @ 54.4 from Beacon data, reverted to MZV @ 42.0 from LIS2DW data |
| Bed PID | Kp 21.945 / Ki 1.068 / Kd 112.742 | Unchanged, good |
| Extruder PID | Kp 15.688 / Ki 0.850 / Kd 72.362 | Unchanged, good |
| Chamber PID | Kp 100 / Ki 0.3 / Kd 30 | **Initial estimates, not measured** — Adam doesn't want PID_CALIBRATE on this. Hand-tune if needed |
| `[output_pin relay]` | `value: 1`, no `shutdown_value` (default 0) | **Intentional safety behaviour** — relay drops on Klipper shutdown, hardware fail-safe. See `memory/relay_trip_gotcha.md` |
| `[beacon] accel_axes_map` | `-x, -y, z` | Added this session per AXES_MAP_CALIBRATION (100% confidence) |
| `[lis2dw ebb] axes_map` | `x, -z, y` | Added this session per AXES_MAP_CALIBRATION |
| `[resonance_tester] accel_chip` | `lis2dw ebb` | Switched from `beacon` this session — Beacon stays as Z probe, LIS2DW for resonance |

---

## Klipper Estimator — what to do in OrcaSlicer

The binary is installed and tested. The static config file is at `/Users/user/klipper_estimator.cfg` on the Mac. To activate it for slicing:

1. Open **OrcaSlicer → Printer Settings → Machine gcode → Post-processing scripts**
2. Paste this single line:

```
/Users/user/klipper_estimator --config_file /Users/user/klipper_estimator.cfg post-process
```

3. Save the printer profile

After that, every slice will be post-processed to overwrite the slicer's optimistic time estimate with an accurate one (typically within ±5 seconds over a multi-hour print). Mainsail progress bars will then be trustworthy.

**Do not** use the `--config_moonraker_url` approach — Adam confirmed it's flaky on this Mac (probably DNS/network/timeout related during slice time when the slicer blocks waiting for Moonraker). The static `--config_file` approach is the one that works.

The static cfg will go stale only if printer motion limits change (`max_velocity`, `max_accel`, `square_corner_velocity`). To re-dump (only when actually needed):

```
ssh user@192.168.x.x "~/klipper_estimator --config_moonraker_url http://localhost:7125 dump-config" > /Users/user/klipper_estimator.cfg
```

(The dump runs on the printer where Moonraker is local and reliable — the flakiness is the Mac↔printer network path during slice time, not the printer itself.)

Current binary version: v3.7.3 (2024-04-27, latest as of 2026-04-10).

---

## Completed in the 2026-04-11 slicer tuning session

Four test print iterations on a 40 mm OD / 30 mm ID annulus, plus a full profile cleanup, established the following:

**Won**:
- **Ironing**: switched from rectilinear (drag marks) to concentric — marks gone, surface clean. Concentric is the default; per-object overrides can select rectilinear for rectangular top surfaces
- **Full settings cleanup**: retraction, Spiral Lift z_hop @ 0.2 mm with Top Only enforce, wipe distance 1 mm, ERS slope 150, `ensure_vertical_shell_thickness: ensure_moderate`, outer wall line width 150%, top/bottom shell layers 5, wall_loops 4, arc fitting disabled (see `2026-04-11-arc-fitting-correction.md` for why)
- **Sunlu PLA Matte filament profile corrected**: vendor, density 1.24, vitrification 55, `filament_retract_restart_extra: 0.04`, `filament_max_volumetric_speed: 24`, z_hop/wipe overrides stripped so printer-level values apply
- **Full round-by-round backups + changelogs** at `orca_slicer/backups/2026-04-11-*` (9 rounds) and `orca_slicer/changelog/2026-04-11-*` (7 changelog entries with rationale, photos, fallback plans)

**Partial — paused at acceptable-not-perfect**:
- **Scarf seam**: ~20 visible step bands still present (round 1's ERS 300→150 softened them but didn't eliminate). Scarf ridge at entry reduced but not fully confirmed gone
- **Traditional seam**: slightly pronounced at `filament_retract_restart_extra: 0.04`. Was "pretty much perfect" at 0.05, minor regression from the v2 blob-reduction

**Not done (future session)**:
- Eliminating the residual scarf banding. Candidate next levers in `orca_slicer/changelog/2026-04-11-scarf-ridge-v2.md` → "Candidate next levers" section. Top candidates: reduce `seam_slope_steps` 20 → 10, or re-calibrate `pressure_advance` (stale since the 2026-04-10 shaper retune)

## First concrete action for the next session

**No blocking next action** — the printer is ready for real print work. Optional priorities if you want to push further:

1. **Print something useful** — the best test of accumulated tuning is real-world print output. Adam explicitly said the seam state is "good enough for now"
2. **Another seam tuning pass** (only if you care, not urgent) — fresh test print, start by experimenting with `seam_slope_steps: 20 → 10` and consider re-calibrating `pressure_advance` since it's been stale since the 2026-04-10 shaper retune. Full candidate lever list in the scarf v2 changelog
3. **KAMP Smart Park + Line Purge** — touchless quality-of-life upgrades (prevents nozzle ooze on print start, adaptive purge line). Each ~10 minutes of config work
4. **Other-material profiles** (ABS/ASA/PETG) — apply the proven seam/ironing settings to sibling process profiles if you want to print those materials. Low priority if PLA covers most of your work
5. **Printer base damping** — sorbothane/cork pads under the cabinet. Likely more print-quality improvement per dollar than any further software tuning

Don't start any of these proactively. Wait for Adam to pick one.

---

## Lessons learned 2026-04-10 (read these before suggesting tuning steps)

These are saved as memory entries; future-Claude should already have them in context. Listing here for human readers and as a backup.

1. **Don't push tuning when there's no concrete problem.** TMC autotune was recommended on the basis of a cosmetic Shake&Tune warning ("motors have different TMC configurations") and made vibrations measurably worse (polar symmetry 71→51%). Reverted. The Klipper docs say *"For most users, no special TMC tuning is required."* Always ask "what problem are we solving?" before recommending a tuning step. NEXT_STEPS.md is a menu of options, not a checklist. (`memory/dont_push_unneeded_tuning.md`)

2. **Don't claim hypotheses as facts.** I had a plausible explanation for the Beacon's bimodal Y peak (mount flex on the cooling-horn duct) and started writing it as established fact. Multiple alternative explanations remain in play (LIS2DW Nyquist limit, sensor characteristics, different physical mounting positions). The currently-saved Y shaper value is a *defensible choice on geometric grounds*, not a *proven correct value*. (`memory/beacon_vs_lis2dw_uncertainty.md`)

3. **Don't add `shutdown_value: 1` to the BTT safety relay.** Adam keeps the default `shutdown_value=0` (relay drops on Klipper shutdown) for hardware fail-safe. Recovery from a relay-trip is a Pi reboot (or possibly `systemctl restart klipper` via Moonraker — half-validated, would benefit from a clean test next session). (`memory/relay_trip_gotcha.md`)

4. **Don't propose chamber heater PID calibration.** Takes 45-60 min to warm chamber, current values work, can be hand-tuned. (`memory/chamber_pid_skip.md`)

5. **Run-to-run measurement variance on Shake&Tune is ~5-10 %.** Don't chase fractional-percent improvements. Don't recommend re-running tests just to "confirm" small numerical changes.

6. **Driver tuning vs print-quality tuning are not the same thing.** TMC autotune optimizes for stepper electrical performance. Input shaper / belt symmetry / probe calibration optimize for print quality. They usually align — but on this build (heavy CNC toolhead, low intrinsic gantry damping) they don't. When recommending any "tuning" step, be clear about which goal it serves.

---

## Open / deferred items

Things on the "could do later" list but not priorities:

- **Validate the Beacon-vs-LIS2DW interpretation properly** — tap test, sensor relocation, third sensor, time-domain comparison. Recorded in `memory/beacon_vs_lis2dw_uncertainty.md`. **Adam said: never going to do this.** Skip unless a specific reason to revisit
- **KAMP Smart Park enablement** — would prevent nozzle ooze landing on the print start area during heat-up. Touchless quality-of-life improvement
- **KAMP Line Purge** — adaptive purge line that traces the print's first layer perimeter. Replaces the existing fixed `PRIME_LINE` macro. Touchless quality-of-life improvement
- **Test the relay-trip recovery via `systemctl restart klipper`** — half-validated 2026-04-10 (services/restart spawned a fresh Klipper process on the same Pi boot, but Adam manually rebooted before polling could confirm post-restart state). Worth a clean re-test if the relay ever trips again
- **Klipper Estimator with static config file** — currently using `--config_moonraker_url`, which requires the Mac to be online to the printer at slice time. If offline slicing becomes important, copy `klipper_estimator.cfg` from the printer to the Mac and switch to `--config_file`
- **Beacon Contact mode + thermal expansion compensation** — bigger change, separate session, useful if first-layer squish varies with temperature
- **Persistent journald on the Pi** — currently `Storage=volatile`, which means we lose pre-reboot kernel logs. Would help next time something low-level crashes the host
- **CAN bus conversion finish** — Katapult is installed on the EBB36, but `can0` is not configured on the Pi. Running over USB for now. Separate project
- **Printer base damping** — Adam's printer sits on a wobbly filing cabinet. Sorbothane/cork pads under the cabinet feet (or under the printer itself) would damp the gantry's intrinsic low-damping resonances and likely improve real print quality more than any further software tuning. ~$15 + 10 minutes once parts arrive
- **Residual scarf step banding (2026-04-11)** — ~20 visible step bands still show on scarf seams even after the v2 restart-extra fix. Round 1's ERS slope reduction (300 → 150) softened them but didn't eliminate. Top candidate next levers: reduce `seam_slope_steps: 20 → 10`, re-calibrate `pressure_advance` (stale since 2026-04-10 shaper retune), or revert `filament_retract_restart_extra: 0.04 → 0.05` to restore perfect normal seams and accept the tradeoff. Full list in `orca_slicer/changelog/2026-04-11-scarf-ridge-v2.md`
- **Pressure advance re-calibration post-shaper-retune** — current value `0.045` was calibrated before the 2026-04-10 MZV shaper change. Not urgent but possibly the root cause of the residual scarf banding and would be the cleanest single-variable experiment if the user picks seam tuning back up
- **Apply proven slicer settings to ABS/ASA/PETG profiles** — the 2026-04-11 session only covered PLA. Other materials would need sibling process profiles plus material-specific filament profiles. Low priority if PLA covers 90% of print work
- **Ironing pattern strategy for non-circular geometries** — concentric works great on round/annular parts but will show visible rings on rectangular top surfaces. Decision per-model via Orca's object-level overrides. Worth documenting a quick "shape → pattern" table in `orca_slicer/process/ironing_deep_dive.md`

---

## File pointers (where things live)

### In the repo

- **`HARDWARE.md`** — hardware reference (corrected 2026-04-10 to remove wrong claims about Y sensorless homing)
- **`docs/references.md`** — best-practices library
- **`tuning/2026-04-10/`** — full Shake&Tune baseline session: 14 PNGs, README with 7 sections, lessons learned
- **`backups/2026-04-10/`** — pre-plugin config snapshot (printer.cfg history, system_info.txt, logs)
- **`orca_slicer/backups/2026-04-10/`** — OrcaSlicer profile snapshot from pre-2026-04-11-session
- **`orca_slicer/backups/2026-04-11-*/`** — per-round profile snapshots from the 2026-04-11 tuning session (9 rounds, each rollback-ready)
- **`orca_slicer/changelog/2026-04-11-*.md`** — changelog entries for each tuning round with rationale, photos, and fallback plans
- **`orca_slicer/process/seams_deep_dive.md`** — 65-reference research doc on seam types, scarf mechanics, and common issues
- **`orca_slicer/process/ironing_deep_dive.md`** — 22-reference research doc on ironing patterns, contiguity, and flow percentages
- **`orca_slicer/filament/sunlu_pla_matte.md`** — SUNLU PLA Matte spec sheet with community gotchas
- **`orca_slicer/printer/`, `orca_slicer/process/`, `orca_slicer/filament/`** — per-setting walkthrough docs (basic_information.md, motion_ability.md, extruder.md, machine_gcode.md, quality_PLA.md, strength_PLA.md, speed_PLA.md, others_PLA.md, seam_diagnosis_field_notes.md)
- **`.mcp.json`** — local `mcp-3d-printer-server` config, now gitignored (contains the real printer IP for local use). Add back manually if you clone this repo fresh on a new machine

### Memory (outside repo, persists across sessions)

Claude Code's auto-memory system — path is environment-specific (derived from the working directory at `~/.claude/projects/<path-encoded>/memory/`). Claude accesses it through internal tools, not by navigating these paths directly. Current memory entries cover: relay trip gotcha, chamber PID skip, don't push unneeded tuning, Beacon-vs-LIS2DW uncertainty, and Klipper Estimator static config preference.

### On the printer

- `~/printer_data/config/printer.cfg` — main config (with this session's changes: `accel_axes_map`, updated SAVE_CONFIG `[input_shaper]`)
- `~/printer_data/config/ebb_gen2.cfg` — EBB36 toolhead config (with `[lis2dw ebb]` enabled)
- `~/printer_data/config/printer.cfg.bak-pre-*` — backups from each edit this session, in case rollback is needed
- `~/printer_data/config/.pre-plugin-backup-20260410/` — snapshot from before the Shake&Tune install
- `~/printer_data/config/KAMP_Settings.cfg` — KAMP variables (all sub-includes still commented out)
- `~/printer_data/config/ShakeTune_results/{axes_map,belts,input_shaper,vibrations}/` — all session PNGs (last 10 results retained per directory)
- `~/klipper_estimator` — printer-side klipper_estimator binary (v3.7.3, ARM)
- `~/klipper_estimator.cfg` — dumped config snapshot (for offline / static-config use)

### On the Mac

- `/Users/user/klipper_estimator` — Mac-side klipper_estimator binary (v3.7.3, x86_64 via Rosetta 2)
- `/Users/user/Repos/3d-printer/` — this repo

### Key connection details

- Printer SSH: `ssh user@192.168.x.x` (key-based, no password)
- `sudo` requires password — non-interactive ssh-via-script for `sudo reboot` does NOT work; use Moonraker's `POST /machine/reboot` or run interactively
- Moonraker API: `http://192.168.x.x:7125`
- Mainsail web UI: `http://192.168.x.x/`
- KlipperScreen: Waveshare 5" DSI on the Pi
- Mac IP (trusted by Moonraker): 192.168.x.x

---

## What's NOT in scope

Things deliberately not being pursued:

- **TMC Autotune** — see lessons learned. Don't recommend
- **Chamber heater PID calibration** — see memory file. Don't recommend
- **Sensorless homing** — Adam doesn't use it; both X and Y use physical endstops
- **Beacon vs LIS2DW validation tests** (tap test, sensor relocation, etc.) — Adam said never going to do this
- **CAN bus conversion** — currently over USB, separate project
- **Klippain framework migration** — looked at, decided against; current config works
- **Klipper-Backup automation** — manual into this repo for now
- **Door interlock / safety upgrades** — worth doing, not blocking tuning work
- **Hardware upgrades** — no nozzle changes, no toolhead replacement, no Beacon-TAP equivalent

---

*End of resume point. No blocking next action — mechanical (2026-04-10) and slicer (2026-04-11) tuning are both at a "good enough for production" state. The residual seam banding is a known, documented, accepted imperfection. See the phase status table and the "First concrete action" section above for optional follow-ups if the user wants to push further.*
