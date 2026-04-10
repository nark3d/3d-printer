# Next steps — resume point after 2026-04-10 tuning session

> This file is the resume point for whoever picks up the project next (you, or future-Claude). Read this first. It supersedes the pre-session version of this doc.
>
> **Current position**: Phase 1 (mechanical sanity) and Phase 3.1 (vibrations profile) are complete. Klipper Estimator is installed. The printer is in genuinely good mechanical shape with documented baseline measurements. **The next concrete action is to run a real calibration test print** with the existing SUNLU PLA Matte profile to validate the tuning end-to-end on actual print quality.

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
| 5.3 | Klipper Estimator install | ✅ Done — binary on Mac at `/Users/adamlewis/klipper_estimator`, also on printer at `~/klipper_estimator`. Orca post-processing line is in the next section |
| 5.4 | Beacon Contact mode + thermal expansion compensation | ⏳ Deferred — bigger change worth its own session if first-layer squish varies between prints |

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

The binary is installed and tested. To activate it for slicing:

1. Open **OrcaSlicer → Printer Settings → Machine gcode → Post-processing scripts**
2. Paste this single line:

```
/Users/adamlewis/klipper_estimator --config_moonraker_url http://192.168.0.37:7125 post-process
```

3. Save the printer profile

After that, every slice will be post-processed to overwrite the slicer's optimistic time estimate with an accurate one (typically within ±5 seconds over a multi-hour print). Mainsail progress bars will then be trustworthy.

Caveats:
- Requires the Mac to reach the printer's Moonraker at slice time. If you're slicing offline, switch to a static config: `--config_file ~/klipper_estimator.cfg` (the file is at `~/klipper_estimator.cfg` on the printer; we'd need to copy it to the Mac for offline use)
- Current binary version: v3.7.3 (2024-04-27, latest as of 2026-04-10)

---

## First concrete action for the next session

**Run a real calibration test print** with the SUNLU PLA Matte profile that Adam has already tuned. The point is to validate the input shaper, Z-tilt, belt symmetry, and probe accuracy *on actual print output*, not on more synthetic measurements.

Suggestions, in order of value:

1. **Voron Tap test print** or any short XY ringing test cube (e.g., a 50 mm hollow cube at the printer's typical print speeds) — verifies the input shaper is correctly tuned by looking at outer-wall ringing
2. **Calibration tower** for any one variable that wasn't covered in Adam's prior calibration
3. **A real print** of something useful — the best test of tuning is whether real prints come out well

Don't run more synthetic tuning macros unless a real print exposes a specific issue.

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

---

## File pointers (where things live)

### In the repo

- **`HARDWARE.md`** — hardware reference (corrected 2026-04-10 to remove wrong claims about Y sensorless homing)
- **`docs/references.md`** — best-practices library
- **`tuning/2026-04-10/`** — full Shake&Tune baseline session: 14 PNGs, README with 7 sections, lessons learned
- **`backups/2026-04-10/`** — pre-plugin config snapshot (printer.cfg history, system_info.txt, logs)
- **`orca_slicer/backups/2026-04-10/`** — OrcaSlicer profile snapshot
- **`.mcp.json`** — `mcp-3d-printer-server` config for the printer at 192.168.0.37

### Memory (outside repo, persists across sessions)

- `~/.claude/projects/-Users-adamlewis-Repos-3d-printer/memory/relay_trip_gotcha.md`
- `~/.claude/projects/-Users-adamlewis-Repos-3d-printer/memory/chamber_pid_skip.md`
- `~/.claude/projects/-Users-adamlewis-Repos-3d-printer/memory/dont_push_unneeded_tuning.md`
- `~/.claude/projects/-Users-adamlewis-Repos-3d-printer/memory/beacon_vs_lis2dw_uncertainty.md`

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

- `/Users/adamlewis/klipper_estimator` — Mac-side klipper_estimator binary (v3.7.3, x86_64 via Rosetta 2)
- `/Users/adamlewis/Repos/3d-printer/` — this repo

### Key connection details

- Printer SSH: `ssh adam@192.168.0.37` (key-based, no password)
- `sudo` requires password — non-interactive ssh-via-script for `sudo reboot` does NOT work; use Moonraker's `POST /machine/reboot` or run interactively
- Moonraker API: `http://192.168.0.37:7125`
- Mainsail web UI: `http://192.168.0.37/`
- KlipperScreen: Waveshare 5" DSI on the Pi
- Mac IP (trusted by Moonraker): 192.168.0.43

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

*End of resume point. The next concrete action is a real calibration test print with SUNLU PLA Matte to validate the tuning on actual print quality, not more synthetic measurements.*
