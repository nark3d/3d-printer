# Next steps — resume point 2026-04-10

> This file exists so that future-you (or future-Claude-after-a-restart) can pick up exactly where the current session left off without losing context. Read this first after restarting Claude Code.
>
> **Current position**: all groundwork is complete and committed. The next concrete action is **run `COMPARE_BELTS_RESPONSES` on the printer**. Everything else is planned in phases below.

---

## Status as of 2026-04-10

### Repo state

Committed as `d89b66c` ("Pre-project groundwork checkpoint"):

- **`HARDWARE.md`** — complete hardware reference (ZeroG Mercury One.1 + Nebula Pro + Hydra 275 + Electronics Enclosure, Octopus Pro V1.0, EBB36 Gen2 via USB, Beacon RevH, VzBot CNC toolhead, Phaetus Rapido 2 HF, Orbiter 2.5, HotBox chamber heater, BentoBox, pin map, full BOM, known gaps)
- **`docs/references.md`** — 772-line best-practices library, 19 sections (tuning playbook, Klippain, Klipper Estimator, Beacon contact mode, mechanical tuning, input shaper comparison, materials, troubleshooting, firmware flashing, video resources, MCP servers, local gotchas)
- **`backups/2026-04-10/`** — pre-plugin snapshot of printer configs, Moonraker DB, systemd env files, metadata (`system_info.txt`)
- **`backups/2026-04-10/logs/`** — Klipper/Moonraker/KlipperScreen logs (on disk, gitignored — not committed)
- **`orca_slicer/backups/2026-04-10/`** — OrcaSlicer snapshot (profiles, custom system presets, user_backup-v* history, `OrcaSlicer.conf`)
- **`.mcp.json`** — configures `mcp-3d-printer-server` for the printer at 192.168.0.37
- **`.gitignore`** — excludes `.DS_Store`, `.claude/`, `backups/*/logs/`

### Local machine (Mac) state

- **`mcp-3d-printer-server` v1.2.2** installed globally via `npm install -g`
- Binary at `/Users/adamlewis/.nvm/versions/node/v20.19.0/bin/mcp-3d-printer-server`
- Configured via `.mcp.json` at repo root (env: `PRINTER_HOST=192.168.0.37`, `PRINTER_PORT=7125`, `PRINTER_TYPE=klipper`, `NOZZLE_DIAMETER=0.4`, `BED_TYPE=textured_pei`)
- **Not yet active** — Claude Code must be restarted in this directory to load the MCP server. After restart, Claude can query the printer live via Moonraker.

### Printer state (192.168.0.37, `3d-printer`, Pi 5 8 GB, Debian Bookworm)

- **Klipper**: `v0.13.0-540-g57c2e0c96-dirty`, state `ready`, PID 148738 (restarted 14:30 on 2026-04-10)
- **Moonraker**: `v0.10.0-8`, healthy
- **Pre-plugin config backup** at `~/printer_data/config/.pre-plugin-backup-20260410/` — contains pre-change `printer.cfg` and `moonraker.conf` for easy rollback
- No print currently running (last was `wrap_around_plate_0.2mm_PLA_..._6h45m.gcode`, state `complete`)

### Plugins installed on printer during this session

| Plugin | Location | State | Activation |
|---|---|---|---|
| **Shake&Tune** (`klippain-shaketune`) | `~/klippain_shaketune/`, symlinked into `~/klipper/klippy/extras/shaketune` | **Active** — `[shaketune]` in `printer.cfg`, 125 gcode commands registered | Ready to use |
| **Klipper TMC Autotune** | `~/klipper_tmc_autotune/`, symlinked `autotune_tmc.py` / `motor_constants.py` / `motor_database.cfg` into extras | **Dormant** — no `[autotune_tmc stepper_x]` blocks configured yet | Activate per-motor later |
| **KAMP** (Klipper-Adaptive-Meshing-Purging) | `~/Klipper-Adaptive-Meshing-Purging/`, `Configuration/` symlinked as `~/printer_data/config/KAMP`, `KAMP_Settings.cfg` copied into `config/` | **Dormant** — `[include KAMP_Settings.cfg]` is in `printer.cfg`, but all four functional sub-includes inside `KAMP_Settings.cfg` are commented out by default | Uncomment sub-includes to enable specific features |

**Side effect to note**: Shake&Tune's pip requirements downgraded **numpy 2.4.3 → 2.2.2** in the `klippy-env` venv on the printer. Klipper is fine with either. Flag if anything else needs NumPy ≥ 2.4.

### Currently saved tuning values (for comparison later)

| Setting | Value | Source |
|---|---|---|
| `max_velocity` | 600 mm/s | `printer.cfg:235` |
| `max_accel` | 10 000 mm/s² | `printer.cfg:236` |
| Input shaper X | **MZV @ 63.8 Hz** | `printer.cfg:438-439` |
| Input shaper Y | **MZV @ 43.4 Hz** | `printer.cfg:440-441` |
| Extruder PID | Kp 15.688 / Ki 0.850 / Kd 72.362 | `ebb_gen2.cfg:28-31` |
| Bed PID | Kp 21.945 / Ki 1.068 / Kd 112.742 | `printer.cfg:433-435` |
| Chamber PID | Kp 100 / Ki 0.3 / Kd 30 | `printer.cfg:128-130` (**initial estimates, not measured**) |
| X rotation_distance | 39.421 (calibrated) | `printer.cfg:26` |
| Y rotation_distance | 39.581 (calibrated) | `printer.cfg:40` |
| Z rotation_distance | 2.002 / 1.990 / 1.990 (Z / Z1 / Z2, Oldham-calibrated) | `printer.cfg:53 / 65 / 97` |
| X/Y run current | 2.24 A (TMC5160T Pro) | `printer.cfg:249, 259` |
| Z run current | 1.15 A (TMC2209) | `printer.cfg:270, 278, 291` |
| Extruder run current | 0.85 A (TMC2209 on EBB36) | `ebb_gen2.cfg:35` |
| Beacon Y offset | 36 mm (Tentacool/Goliath Short duct) | `printer.cfg:344` |
| Beacon Z calibration model | `model_temp 25.31 °C`, `model_offset −0.09 mm` | `printer.cfg:443-457` |
| Bed mesh | 10×10 bicubic, 5,36 → 255,240 | `printer.cfg:407-430` |

If any Shake&Tune result differs substantially from these, investigate **why** before adopting the new values.

---

## Step 0 — Load the MCP server (you, 30 seconds)

**Restart Claude Code in this directory.** When it comes back up, the `printer` MCP server from `.mcp.json` will load automatically. You'll know it worked because Claude will be able to query Moonraker directly — temperatures, positions, file listings, logs — without needing to SSH or `curl`.

---

## Phase 1 — Mechanical sanity check (read-only, ~15 minutes of printer time)

**Prerequisites**:
- Printer must be **cold** (no active heating). The HotBox, bed, and hotend should all be off and cool. The fans shouldn't be running.
- Printer must be **homed** (`G28`) so the toolhead knows where it is.
- Bed must be **clear** — no print in progress, no objects on the plate. The toolhead will move around quite a bit.
- Your Mac must be on the same network as the printer (192.168.0.43 ↔ 192.168.0.37).

**Goal of Phase 1**: confirm the machine is mechanically sound before spending any time on software tuning. Software tuning on a mechanically bad printer is a waste of time.

### 1.1 — `AXES_MAP_CALIBRATION` (one-time sanity check)

```
AXES_MAP_CALIBRATION
```

Verifies the Beacon's XYZ accelerometer axes are oriented correctly relative to the printer's motion. If this fails, nothing downstream is trustworthy. Run once, expect it to pass. Output goes to `~/printer_data/config/ShakeTune_results/`.

### 1.2 — `COMPARE_BELTS_RESPONSES` (the main diagnostic)

```
COMPARE_BELTS_RESPONSES
```

This is the big one. Shake&Tune drives each CoreXY belt (A and B) independently and plots their frequency response on the same graph. It tells us:

- **Whether the A and B belts are at the same tension** — if so, both curves overlap
- **Whether one belt has bearing or pulley issues** — curves will diverge in characteristic patterns
- **Whether there's a frequency delta** (one belt tighter than the other) — shows as horizontal offset on the paired-peak points

**How to read the output graph** (reference: [`compare_belts_responses.md`](https://github.com/Frix-x/klippain-shaketune/blob/main/docs/macros/compare_belts_responses.md)):

- The graph has a **good zone** (green shading) that's wider at the bottom (low-energy regions where deviation doesn't matter) and narrower at the top (high-energy regions where the peaks are)
- All data points should be close to the centre line and ideally inside the green zone
- Paired peaks at the same frequency → single point (labelled α1/α2, β1/β2 etc); distance from centre = energy delta
- Paired peaks at different frequencies → two points; distance along the plotted line = frequency delta
- **Prerequisite**: both belts must be the same brand, same batch, same number of teeth, same wear level. A single tooth difference ruins the comparison.

### 1.3 — `AXES_SHAPER_CALIBRATION` (improved input shaper tuning)

```
AXES_SHAPER_CALIBRATION
```

Shake&Tune's replacement for Klipper's built-in `TEST_RESONANCES` + `SHAPER_CALIBRATE`. Better graphs, better recommendations. Compare the suggested values against our current saved values (MZV X=63.8 / Y=43.4).

**Decision rule** based on the graph it produces:

| Graph shows | Use shaper type | Notes |
|---|---|---|
| Single sharp peak per axis | **MZV** | Our current choice — probably stay here |
| Multiple peaks within a few Hz | **EI** | More smoothing than MZV but more forgiving |
| Two peaks separated by >5 Hz | **2HUMP_EI** | `shaper_freq` at the lower peak |
| Very low peak (< 25 Hz) | **STOP** | Mechanical problem first. Toolhead too heavy, frame too flexible, or input shaping won't save you. |

See `docs/references.md` → *Input shaper: shaper type comparison and workflow* for full details.

### How Claude will fetch results

After the MCP server is loaded, Claude can pull the PNGs and interpret them. The results land in `~/printer_data/config/ShakeTune_results/` on the printer — either visible in the Mainsail file browser (because `show_macros_in_webui: True` is the Shake&Tune default) or fetchable via the MCP file_manager API. Without the MCP loaded, Claude can still `scp` or `rsync` them down to the Mac manually.

---

## Phase 2 — Decision point (you interpret, Claude advises)

Based on Phase 1 results, one of three paths:

### 2.A — All green: belts symmetric, input shaper consistent

**Machine is mechanically healthy.** Proceed to Phase 3.

### 2.B — Belts asymmetric

Physical retensioning required. **This is not a software fix.** Claude cannot do this remotely. You'll need:

1. A screwdriver to loosen/tighten the belt tensioners
2. **Gates Carbon Drive app** (free on iOS/Android) or a guitar-tuner app to measure belt frequency by plucking each belt
3. Target: 50–60 Hz on a typical CoreXY, BUT — **the primary goal is squareness, not absolute frequency.** Adjust one belt until `COMPARE_BELTS_RESPONSES` shows matching curves; that's the right tension.
4. After physical retensioning, re-home (`G28`) and re-run `COMPARE_BELTS_RESPONSES` to verify.

Reference: `docs/references.md` → *Mechanical tuning: rails, belts, frame squaring*.

### 2.C — Input shaper values moved significantly from current

**Investigate why before adopting new values.** Possible causes:

- Mass changed on the toolhead since last tune (nozzle swap, sock change, wiring rerouted)
- Belt wear has altered the resonance
- Frame bolts loose somewhere → lower resonant frequency
- Toolhead screws loose → higher or lower depending on what's rattling
- PTFE tube routing changed, creating a drag point
- Belt tension drifted (see 2.B)

Don't blindly copy the new values. Identify the change, fix the cause if it's bad, then re-measure.

---

## Phase 3 — Refinement tuning (only after Phase 1 passes)

### 3.1 — `CREATE_VIBRATIONS_PROFILE`

```
CREATE_VIBRATIONS_PROFILE
```

The deep resonance map. Sweeps the toolhead through every direction (multiple angles) and every speed (from low to `max_velocity`). Produces a heatmap of vibration by direction × speed. Tells us things like "your printer rings badly at 180 mm/s on the 45° diagonal but is fine at 200 mm/s on pure X" — information you can't get from any other test.

Takes longer than the previous tests (15–30 minutes depending on the speed range). Still printer-cold.

### 3.2 — PID recalibration if heaters drift

Only if Phase 1 exposed temperature instability or you notice the chamber heater overshooting:

```
PID_CALIBRATE HEATER=chamber TARGET=50
SAVE_CONFIG
```

The chamber PID values in the current config (Kp 100 / Ki 0.3 / Kd 30) are **initial estimates, not measured**. These were never formally tuned. Worth re-running before relying on chamber heating for production prints.

Hotend and bed PIDs are probably fine (they were tuned and haven't changed).

### 3.3 — Klipper TMC Autotune enablement (judgment call)

Adding `[autotune_tmc stepper_x]` blocks per motor. This is the step I (Claude) should not take alone — it changes current, TBL/HEND/HSTRT/TOFF parameters, and potentially the noise/heat characteristics of the motors.

**When you're ready to enable it**, the workflow is:

1. Look up the motor name in `~/klipper_tmc_autotune/motor_database.cfg` — e.g., `ldo-42sth48-2804ac` for the X/Y motors
2. If it's not in the database, add a custom entry via `[motor_constants custom_name]` in `printer.cfg`
3. Add per-motor autotune blocks:
   ```
   [autotune_tmc stepper_x]
   motor: ldo-42sth48-2804ac
   tuning_goal: auto   # or 'silent' / 'performance'
   
   [autotune_tmc stepper_y]
   motor: ldo-42sth48-2804ac
   tuning_goal: auto
   
   [autotune_tmc stepper_z]
   motor: <whatever-the-database-name-for-wotrees-17hs4401-is>
   tuning_goal: auto
   # Same for stepper_z1, stepper_z2
   ```
4. Add `[autotune_tmc extruder]` in `ebb_gen2.cfg` — motor is the Orbiter 2.5 stock LDO pancake
5. `RESTART` Klipper and check klippy.log for any complaints
6. Re-run `COMPARE_BELTS_RESPONSES` — if the autotune changed current distribution, belts may need recheck

Claude can prepare these edits for your review; you approve before they're applied.

---

## Phase 4 — Per-filament slicer calibration

Re-run Orca's calibration sequence **per new filament roll**, not per profile. A fresh spool of "the same filament" from the same vendor can still vary.

Order:

1. **Temperature tower** — find the lowest temp that gives good bridging without under-extrusion
2. **Flow rate** — coarse then fine
3. **Pressure advance** — after flow rate is stable
4. **Max volumetric speed** — find the point where surface quality breaks down
5. **Retraction** — usually last

Known issue to resolve at some point: the `eSun ASA Plus` profile has `max_volumetric_speed: 10` while `eSun ASA Plus - Clean` has 18 — one of these was measured, the other is stale. Re-run volumetric speed calibration on whichever roll is current to resolve the conflict.

---

## Phase 5 — Adaptive features (quality-of-life upgrades)

Only after Phases 1-4 are solid. Each one is optional and can be enabled independently.

### 5.1 — Enable KAMP adaptive meshing

Edit `~/printer_data/config/KAMP_Settings.cfg` and uncomment:

```
[include ./KAMP/Adaptive_Meshing.cfg]
```

Then edit `~/printer_data/config/moonraker.conf` and change:

```
[file_manager]
enable_object_processing: False
```

to:

```
[file_manager]
enable_object_processing: True
```

(This is required because adaptive meshing needs to know which bed area the print touches, which Moonraker computes from the gcode's object boundaries.)

Restart Klipper and Moonraker. After this, `BED_MESH_CALIBRATE ADAPTIVE=1` (which your `PRINT_START` already calls) will mesh only the area the print uses. Saves significant time on small prints.

### 5.2 — Enable KAMP Smart Park

In `KAMP_Settings.cfg`, uncomment:

```
[include ./KAMP/Smart_Park.cfg]
```

Then update your `PRINT_START` macro in `macros.cfg` to call `SMART_PARK` after `BED_MESH_CALIBRATE` and before the final `M109`. The toolhead will park over the print origin at a safe Z during the final heat-up, preventing nozzle ooze landing on your finished prints.

### 5.3 — Install Klipper Estimator

Post-processes slicer gcode to replace optimistic print time estimates with accurate ones (±5 seconds over a multi-hour print). Mainsail progress bars become trustworthy.

Install steps are in [the freakyDude blog post](https://blog.freakydu.de/posts/2025-09-05-improved_klipper_print_estimation/). Summary:

1. Download `klipper_estimator` binary from [github.com/Annex-Engineering/klipper_estimator/releases](https://github.com/Annex-Engineering/klipper_estimator/releases) (ARM64 for Pi 5)
2. Place at `~/klipper_estimator` on the printer
3. Fetch the printer's config snapshot: `klipper_estimator --config_moonraker_url http://localhost:7125 dump-config > ~/klipper_estimator.cfg`
4. Add to Orca's *Printer → Machine gcode → Post-processing scripts*: `~/klipper_estimator --config_file ~/klipper_estimator.cfg post-process`
5. Re-slice anything — print time estimates will now match reality

This is the **biggest quality-of-life upgrade** in the whole plan. Low risk, immediate benefit.

### 5.4 — Beacon Contact mode + thermal expansion compensation

If you notice first-layer squish varying between prints (especially with temperature changes), this is the fix. Requires installing [`YanceyA/BeaconPrinterTools`](https://github.com/YanceyA/BeaconPrinterTools):

1. `git clone https://github.com/YanceyA/BeaconPrinterTools.git ~/BeaconPrinterTools`
2. Follow the `Thermal_Expansion_Compensation/Thermal_expansion_compensation.md` README
3. Add Beacon contact parameters to the `[beacon]` section of `printer.cfg`
4. Run `BEACON_CALIBRATE_NOZZLE_TEMP_OFFSET` once to derive the expansion coefficient
5. Add the thermal compensation macro call to your `PRINT_START`

After this, Beacon automatically applies the right Z offset for the nozzle temperature of each print — no more babystepping per filament type. Bigger change than the others, worth its own session.

---

## Safety reminders for Phases 1–3

- **Printer cold** — heaters off, chamber off, fans off. Resonance testing moves the toolhead aggressively; you don't want anything thermal happening at the same time.
- **Bed clear** — no print, no parts, no tools on the plate. The toolhead will traverse the full envelope during vibration profiles.
- **Homed** — `G28` before any Shake&Tune macro. The macros don't home automatically.
- **Claude will ask before any command that writes to state** — the Moonraker auth allows writes from our subnet, but any gcode that isn't purely read-only (e.g., setting temps, starting prints, `SAVE_CONFIG`) will be confirmed first.

---

## First concrete action after Claude Code restart

The very first command after restart:

```
G28
AXES_MAP_CALIBRATION
```

Wait for the macro to finish (it'll move the toolhead in a controlled pattern and output axis orientation data). Then:

```
COMPARE_BELTS_RESPONSES
```

That's the baseline. Everything else is decided after seeing the output of that one command.

---

## File pointers (where things live)

### In the repo

- `HARDWARE.md` — hardware reference, with config line numbers
- `docs/references.md` — the best-practices library
- `backups/2026-04-10/config/printer.cfg` — **pre-plugin** snapshot (before `[shaketune]` and `[include KAMP_Settings.cfg]` were added)
- `backups/2026-04-10/config/ebb_gen2.cfg` — EBB36 toolhead config
- `backups/2026-04-10/system_info.txt` — full system snapshot (versions, services, USB, venvs, dmesg)
- `backups/2026-04-10/logs/klippy.log` — 64 MB klippy log (gitignored, local only)
- `.mcp.json` — MCP server configuration

### On the printer

- `~/printer_data/config/printer.cfg` — **live** config (has `[shaketune]` and `[include KAMP_Settings.cfg]`)
- `~/printer_data/config/.pre-plugin-backup-20260410/` — snapshot taken before plugin install
- `~/printer_data/config/KAMP_Settings.cfg` — KAMP variables (all sub-includes commented out)
- `~/printer_data/config/KAMP` → symlink to `~/Klipper-Adaptive-Meshing-Purging/Configuration/`
- `~/printer_data/config/ShakeTune_results/` — **where Shake&Tune output lands** (will exist after first macro run)
- `~/klippain_shaketune/` — Shake&Tune source
- `~/klipper_tmc_autotune/` — TMC Autotune source (dormant)
- `~/Klipper-Adaptive-Meshing-Purging/` — KAMP source (dormant)
- `~/printer_data/logs/klippy.log` — live klippy log
- `~/printer_data/logs/moonraker.log` — live moonraker log

### Key connection details

- Printer SSH: `ssh adam@192.168.0.37` (key-based, no password)
- Moonraker API: `http://192.168.0.37:7125`
- Mainsail web UI: `http://192.168.0.37/`
- KlipperScreen: Waveshare 5" DSI on the Pi
- Mac IP (trusted by Moonraker): 192.168.0.43

---

## What's NOT in scope for this next session

Things we deliberately aren't doing yet, for reference:

- **CAN bus conversion finish** — Katapult is installed on the EBB36, but `can0` is not configured on the Pi. Running over USB for now. Separate project.
- **Klippain framework migration** — we looked at it, decided it's a re-platforming cost we don't need to pay. Current config works.
- **Klipper-Backup automation** — we're doing it manually into this repo. Could automate it later.
- **Safety upgrades** — no door interlock, no exhaust fan, no fire alarm verification. Worth addressing but not blocking tuning work.
- **Hardware upgrades** — no nozzle changes planned, no toolhead replacement, no Beacon-TAP equivalent.
- **What the project actually is** — Adam hasn't told me yet. He said "we must do the groundwork first" and this doc is the final groundwork artifact.

---

*End of resume point. After Claude Code restart, open this file first, verify the MCP server is loaded (you'll see `printer` in the MCP server list), and start with the `G28` + `AXES_MAP_CALIBRATION` + `COMPARE_BELTS_RESPONSES` sequence.*
