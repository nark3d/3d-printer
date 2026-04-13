# Klipper Config Cleanup & Reorganisation Plan

> Research and plan document for tidying up the Klipper config directory. **Plan only — nothing to be applied until user approves.** Written 2026-04-12 while a print is running.

---

## Table of contents

1. [Executive summary](#1-executive-summary)
2. [Current state audit](#2-current-state-audit)
3. [Community best practice — what the research says](#3-community-best-practice--what-the-research-says)
4. [Recommended target structure](#4-recommended-target-structure)
5. [Cleanup categories — what needs doing](#5-cleanup-categories--what-needs-doing)
6. [Phased migration plan (low-risk ordering)](#6-phased-migration-plan-low-risk-ordering)
7. [Ongoing maintenance practices](#7-ongoing-maintenance-practices)
8. [References](#8-references)

---

## 1. Executive summary

**Our config is in reasonable shape** — the `printer.cfg + [include]` shell pattern is already in use, the toolhead (EBB36) is already a separate file, web-UI configs (`mainsail.cfg`) are not directly edited, and the most important structural choices match community best practice.

**What needs work**:
- **~65 lines of dead code** across printer.cfg, macros.cfg, ebb_gen2.cfg — commented-out blocks for hardware that moved to the EBB36 a while back. The Voron community considers leaving these in production configs an anti-pattern [22][24].
- **~75 lines of HOTBOX config** sprawling in printer.cfg when it should live in `hotbox_macros.cfg` or a new `chamber.cfg`. Aligns with the functional-decomposition pattern [12].
- **44 accumulated `printer-YYYYMMDD_HHMMSS.cfg` auto-backups** — safe to delete, should be `.gitignored` per the canonical Klipper-Backup exclusion list [27][28].
- **3 potential duplicate section definitions** between `printer.cfg` and `mainsail.cfg` (`[virtual_sdcard]`, `[pause_resume]`, `[display_status]`). May be silently tolerated by Klipper but worth cleaning up.
- **Stepper configs separated from driver configs** by 200+ lines in printer.cfg — minor taste issue, could be co-located or split into per-axis files.

**What to NOT do**:
- **Don't adopt Klippain or RatOS** — overkill for a single-printer setup with fully-custom hardware. Klippain is brilliant if you're running Voron-standard hardware or multiple printers with shared config, but it adds abstraction overhead we don't need [5][6][16].
- **Don't try to move the SAVE_CONFIG `#*#` block** — Klipper enforces it must be at the end of `printer.cfg` specifically. There's an open feature request from years ago, no progress [19].
- **Don't edit `mainsail.cfg`, `timelapse.cfg`, or `moonraker_obico_macros.cfg`** — they're symlinks to auto-updated content, changes will be lost.

**Total estimated impact**:
- printer.cfg: 478 lines → ~310 lines (-35%)
- macros.cfg: 265 lines → ~240 lines (-10%, TEST_SPEED is 120 of those lines)
- hotbox_macros.cfg: 90 lines → ~160 lines (+78%, absorbing the HOTBOX config from printer.cfg)
- New file: `chamber.cfg` or extended `hotbox_macros.cfg` with the sensor/heater/fan definitions
- `.gitignore`: add `printer-[0-9]*_[0-9]*.cfg`

---

## 2. Current state audit

### Files on the printer (`~/printer_data/config/`)

| File | Size | Status | Notes |
|---|---|---|---|
| `printer.cfg` | 11.2 KB | **Needs cleanup** | Commented-out blocks, HOTBOX sprawl, stepper/driver separation |
| `ebb_gen2.cfg` | 1.9 KB | Minor cleanup | One commented-out `[resonance_tester]` block at bottom |
| `macros.cfg` | 10.2 KB | **Needs cleanup** | Dead `PRIME_LINE` + empty `FANS_ON` + lots of `TEST_SPEED` |
| `led_macros.cfg` | 7.2 KB | Clean | Rewritten 2026-04-12 |
| `hotbox_macros.cfg` | 4.2 KB | Clean (but will absorb config) | Well-structured; would grow by ~50% |
| `KAMP_Settings.cfg` | 3.0 KB | Clean | Two includes enabled, two commented (correct) |
| `mainsail.cfg` | 16.5 KB | **Do not edit** | Symlink to `~/mainsail-config/` (auto-updated) |
| `moonraker_obico_macros.cfg` | 8.4 KB | **Do not edit** | Symlink to Obico install |
| `timelapse.cfg` | 22.5 KB | **Do not edit** | Symlink to moonraker-timelapse install |
| `moonraker.conf` | 4.1 KB | Minor cleanup | Commented-out `[timelapse]` / `[update_manager timelapse]` block + dead camera sections |
| `moonraker-obico.cfg` | 255 B | **Contains secret** | New post-rotation auth_token — exclude from any backup sync |
| `moonraker-obico-update.cfg` | 310 B | Clean | |
| `crowsnest.conf` | 2.9 KB | Clean | Standard MainsailOS webcam config |
| `sonar.conf` | 771 B | Review | `enable: false` — consider uninstalling if never used |
| `KlipperScreen.conf` | 359 B | Clean | |
| `KAMP/Smart_Park.cfg` etc. | — | **Do not edit** | Third-party plugin files (managed by KAMP update_manager) |
| `printer-YYYYMMDD_HHMMSS.cfg` | × 44 | **Clutter** | Delete + gitignore |

### Dead code inventory

**`printer.cfg` — 7 commented-out blocks** for hardware moved to EBB36 (~60 lines):

| Block | Lines | Status |
|---|---|---|
| `#[extruder]` | 69-85 (17 lines) | Moved to ebb_gen2.cfg |
| `#[filament_switch_sensor material_0]` | 87-88 | Never used |
| `#[filament_switch_sensor material_1/2/3]` | 100-107 (6 lines) | Never used |
| `#[fan]` | 191-193 (3 lines) | Moved to ebb_gen2.cfg |
| `#[heater_fan heatbreak_cooling_fan]` | 195-197 (3 lines) | Moved to ebb_gen2.cfg |
| `#[tmc2209 extruder]` | 282-285 (4 lines) | Moved to ebb_gen2.cfg |
| `#[bltouch]` | 328-332 (5 lines) | Replaced by Beacon |
| `#[adxl345]` | 355-358 (4 lines) | Replaced by LIS2DW on EBB36 |
| Comment mentions "EBB42" | Multiple | Board is actually EBB36 Gen2 |

**`macros.cfg` — 2 dead macros** (~25 lines):
| Macro | Lines | Status |
|---|---|---|
| `PRIME_LINE` | 93-114 (22 lines) | Replaced by `LINE_PURGE` in PRINT_START on 2026-04-12 |
| `FANS_ON` | 50-52 (3 lines) | Body is a single commented-out SET_PIN — effectively empty |

**`ebb_gen2.cfg` — 1 commented-out block** (3 lines):
| Block | Status |
|---|---|
| `#[resonance_tester]` at bottom | Moved to printer.cfg |

**`moonraker.conf` — 2 blocks of dead comments** (~25 lines):
| Block | Status |
|---|---|
| `[update_manager timelapse]` / `[timelapse]` commented out | Timelapse is actually installed and symlinked |
| `[camera]` / `[mjpeg_stream]` commented out | Crowsnest handles this instead |

### Duplicate sections

`mainsail.cfg` (symlinked) defines `[virtual_sdcard]`, `[pause_resume]`, `[display_status]`.

`printer.cfg` ALSO defines the same three sections at lines 406-409:
```
[virtual_sdcard]
path: ~/printer_data/gcodes
[display_status]
[pause_resume]
```

Klipper uses last-definition-wins for duplicate sections [3]. Current order means `printer.cfg`'s definitions override `mainsail.cfg`. The `[virtual_sdcard]` in `printer.cfg` ADDS the `path:` line that mainsail.cfg also sets. Not a functional problem but cosmetic duplication.

### Organisational observations

**printer.cfg layout jumps around**:
```
Lines 20-107:  [stepper_x], [stepper_y], [stepper_z], [stepper_z1], [stepper_z2]
Lines 109-189: [heater_bed], HOTBOX cluster, temperature sensors, enclosure fans
Lines 225-231: MCU definitions ([mcu], [mcu EBB36], include ebb_gen2.cfg)
Lines 233-238: [printer] kinematics
Lines 244-293: TMC driver configs (TMC5160 for X/Y, TMC2209 for Z/Z1/Z2)
Lines 295-309: [board_pins], display
Lines 322-369: Beacon, bed_mesh, resonance_tester, z_tilt
Lines 384-415: idle_timeout, relay, misc Klipper modules, [include] block
Lines 417+:    SAVE_CONFIG auto-generated block
```

The HOTBOX cluster (60 lines) and the TMC driver sections (50 lines) are where the logical separation breaks down.

### Auto-backup accumulation

**44 `printer-YYYYMMDD_HHMMSS.cfg` files** from Mar 1 through Apr 12. Total size ~500KB. These are snapshots of just `printer.cfg` (not any included files) created on every Mainsail edit and every SAVE_CONFIG run [27]. Klipper keeps them forever unless manually deleted.

---

## 3. Community best practice — what the research says

From 40 sources surveyed (see §8 references).

### Core findings (high confidence)

1. **No canonical "best" layout exists** — Klipper's docs prescribe almost nothing beyond "includes work" [1]. Everything else is community-driven.

2. **The only hard constraint is SAVE_CONFIG block location** — must be at the end of `printer.cfg`, nowhere else. Klipper enforces this [1][19].

3. **Two dominant patterns exist**:
   - **Shell + functional includes** (Pattern 1 — most widespread) — `printer.cfg` is thin, with includes for each functional area
   - **Framework + overrides** (Pattern 2 — Klippain/RatOS) — read-only framework files + user-specific overrides

4. **Monolithic printer.cfg is the most common beginner state and the most common anti-pattern** warned against by experienced users [14][15].

5. **Commented-out alternative hardware in production configs is explicitly called out** as an anti-pattern by the Voron community [22][24]. Acceptable in starter templates (to show options), not in production configs. Git history is the right place to preserve "what used to be there."

### Specific practices

6. **Auto-backup files** (`printer-[0-9]*_[0-9]*.cfg`) are safe to delete and should be `.gitignored`. This is the canonical pattern in Klipper-Backup's default `.env` exclude list [28].

7. **Toolhead (CAN/USB) should be a separate file** when using an EBB36 or similar — community consensus across maz0r/klipper_canbus, Klippain, and most Voron configs [31][32][33].

8. **mainsail.cfg / fluidd.cfg should never be edited directly** — they're maintained by the mainsail-config/fluidd-config repos and auto-updated via moonraker's update_manager [17][18]. Customisation goes through `_CLIENT_VARIABLE` in your own config.

9. **Don't use a macro framework (klipper-macros, Klippain) alongside another** — jschuh warns explicitly that "advanced Klipper macros tend to rely extensively on monkey patching" [10] and conflicts are common.

10. **SAVE_CONFIG interaction with overrides is the unresolved problem in framework configs** — even Klippain hasn't fully solved it [8]. For a non-framework setup, this is a non-issue: PID/shaper/mesh values just live in the `#*#` block and you leave them there.

### What does NOT have community consensus

- **Degree of modularity**: some users use one-file-per-macro (rootiest — 485 stars [11]), others use flat functional files (zellneralex — 386 stars [12]). Both are valid.
- **Where to put `[printer]` kinematics and MCU definitions**: some include `printer.cfg` stays minimal with everything in includes; others keep MCU + `[printer]` in `printer.cfg`. Taste-based.
- **Whether to have separate `stepper_x.cfg` per axis, or one `steppers.cfg`, or co-located with driver configs**: zero consensus.
- **Comment density**: ranges from zellneralex's banner-heavy approach to garethky's minimal-inline-comments-plus-separate-readme approach [13]. Taste.

### Specific anti-patterns to avoid (consensus)

1. **Everything in one `printer.cfg`** — hard to navigate, hard to diff, harder to share settings between machines [15]
2. **Large commented-out alternative hardware blocks** in a production config [22]
3. **Editing `mainsail.cfg` directly** — changes will be overwritten on next update [17]
4. **Letting `printer-YYYYMMDD_HHMMSS.cfg` accumulate** — add to gitignore [28]
5. **Mixing macro frameworks** (e.g., Klippain + jschuh + klipper-macros) — overlapping monkey-patching causes conflicts [9][10]

---

## 4. Recommended target structure

### Pattern choice: Shell + flat functional decomposition (Pattern 1)

For a single-printer setup with fully custom hardware, Pattern 1 is the right choice. Pattern 2 (Klippain/RatOS framework) is designed for shared community configs across many printers — overkill here and adds unnecessary abstraction. We keep full control and simplicity.

**Rationale**:
- We already have the basic structure (printer.cfg + functional includes). Migrating to Klippain would be a rewrite, not a cleanup.
- We're not sharing this config across printers
- We customise hardware specifically for this build — the override/abstraction pattern of Klippain fights against that
- SAVE_CONFIG interaction is simpler (just lives in `printer.cfg`'s auto-block)

### Target file layout

```
~/printer_data/config/
├── printer.cfg              ← shell: MCU defs, [printer] kinematics, includes, SAVE_CONFIG block
├── ebb_gen2.cfg             ← toolhead: extruder, fans, endstop, LIS2DW (unchanged)
├── macros.cfg               ← PRINT_START, PRINT_END, M600, RESUME, TURN_RELAY_ON/OFF, TEST_SPEED
├── led_macros.cfg           ← NeoPixel effects + LED macros (already clean)
├── chamber.cfg              ← NEW: [heater_generic chamber], [temperature_fan hotbox],
│                                     HOTBOX_CONTROL delayed_gcode, [output_pin enclosure_fans],
│                                     ENCLOSURE_FAN_CONTROL delayed_gcode, [verify_heater chamber]
├── hotbox_macros.cfg        ← HOTBOX_ON, HOTBOX_OFF, HOTBOX_STATUS, HOTBOX_STATE, HOTBOX_RESTART
│                                     (unchanged — stays macros-only)
├── KAMP_Settings.cfg        ← unchanged
├── mainsail.cfg             ← symlink (don't edit)
├── moonraker_obico_macros.cfg ← symlink (don't edit)
├── timelapse.cfg            ← symlink (don't edit)
├── KAMP/                    ← plugin files (don't edit)
└── (non-printer.cfg configs: moonraker.conf, crowsnest.conf, sonar.conf, KlipperScreen.conf)
```

**Key decisions**:
1. **Split HOTBOX HARDWARE config from HOTBOX MACROS** — hardware goes to `chamber.cfg`, macros stay in `hotbox_macros.cfg`. Parallel to how `ebb_gen2.cfg` has hardware config and `macros.cfg` has the usage macros.
2. **Keep `macros.cfg` as a single file** — not worth splitting into per-macro files for our scale. TEST_SPEED dominates but is self-contained.
3. **Don't create `stepper_x.cfg` per-axis files** — our 5 steppers aren't that different from each other. Co-locate in printer.cfg with drivers (see §5.4).
4. **Don't rename `ebb_gen2.cfg`** — it's clear what it is.

### What printer.cfg should look like after cleanup

```
# Octopus Pro main MCU definitions
[mcu]
serial: /dev/serial/by-id/usb-Klipper_stm32f429xx_...

[mcu EBB36]
serial: /dev/serial/by-id/usb-Klipper_stm32g0b1xx_EBB36_GEN-if00

[include ebb_gen2.cfg]

[printer]
kinematics: corexy
max_velocity: 600
max_accel: 10000
max_z_velocity: 5
max_z_accel: 50

# ===== Steppers (hardware + drivers co-located) =====
[stepper_x]
step_pin: ...
...

[tmc5160 stepper_x]
cs_pin: PC4
...

[stepper_y]
...

[tmc5160 stepper_y]
...

[stepper_z]
...

[tmc2209 stepper_z]
...

[stepper_z1]
...

[tmc2209 stepper_z1]
...

[stepper_z2]
...

[tmc2209 stepper_z2]
...

# ===== Bed =====
[heater_bed]
heater_pin: PA1
...

# ===== Temperature sensors (non-heater) =====
[temperature_sensor Octopus_pro]
[temperature_sensor raspberry_pi]
[temperature_sensor chamber_bottom]

# ===== Outputs (lights, relays, non-heater fans) =====
[output_pin bentobox]
[output_pin main_lights]
[output_pin relay]

# ===== Kinematics and probe =====
[board_pins]
[display]
[output_pin beeper]
[safe_z_home]
[beacon]
[bed_mesh]
[resonance_tester]
[z_tilt]
[idle_timeout]
[force_move]
[gcode_arcs]
[exclude_object]
[firmware_retraction]

# ===== Third-party plugin =====
[shaketune]

# ===== Includes =====
[include mainsail.cfg]          # Mainsail web UI — DO NOT EDIT mainsail.cfg directly
[include led_macros.cfg]        # LED effects + macros
[include chamber.cfg]           # NEW: HotBox heater + enclosure fan control
[include hotbox_macros.cfg]     # HOTBOX_ON / _OFF / _STATUS user macros
[include macros.cfg]            # PRINT_START, PRINT_END, M600, utilities
[include moonraker_obico_macros.cfg]
[include KAMP_Settings.cfg]

#*# <---------------------- SAVE_CONFIG ---------------------->
#*# DO NOT EDIT THIS BLOCK OR BELOW. The contents are auto-generated.
...
```

**Target size**: ~310 lines (down from 478, a 35% reduction).

### What chamber.cfg should contain

All the hotbox hardware stuff currently living in printer.cfg:

```
# ============================================================
# chamber.cfg
# HotBox chamber heater hardware and control loops
# User-facing macros: see hotbox_macros.cfg
# ============================================================

[heater_generic chamber]
heater_pin: PA3
sensor_type: Generic 3950
sensor_pin: PF6
min_temp: 0
max_temp: 70
control: pid
pwm_cycle_time: 0.01
pid_Kp: 100
pid_Ki: 0.3
pid_Kd: 30

[verify_heater chamber]
max_error: 600
check_gain_time: 300
hysteresis: 5
heating_gain: 0.5

[temperature_fan hotbox]
pin: PD14
sensor_type: Generic 3950
sensor_pin: PF7
min_temp: 0
max_temp: 120
target_temp: 30.0
min_speed: 0.0
max_speed: 1.0
control: watermark
max_delta: 1
kick_start_time: 0.5

# Auto-start HotBox control loop: runs every 5s, disables heater if element hits 87°C
[delayed_gcode HOTBOX_CONTROL]
initial_duration: 5
gcode:
    ...

[delayed_gcode HOTBOX_BACKOFF]
...

# Enclosure fans: driven by Pi + Octopus MCU temperatures
[output_pin enclosure_fans]
pin: PD12
pwm: True

[delayed_gcode ENCLOSURE_FAN_CONTROL]
initial_duration: 10
gcode:
    ...
```

---

## 5. Cleanup categories — what needs doing

### 5.1 Dead code removal — `printer.cfg`

**Action**: delete all commented-out alternative hardware blocks.

Blocks to remove (with line numbers as of 2026-04-12):
- `#[extruder]` — lines 69-85 (moved to ebb_gen2.cfg)
- `#[filament_switch_sensor material_0/1/2/3]` — lines 87-88, 100-107 (never used)
- `#[fan]` — lines 191-193 (moved to ebb_gen2.cfg)
- `#[heater_fan heatbreak_cooling_fan]` — lines 195-197 (moved to ebb_gen2.cfg)
- `#[tmc2209 extruder]` — lines 282-285 (moved to ebb_gen2.cfg)
- `#[bltouch]` — lines 328-332 (replaced by Beacon)
- `#[adxl345]` — lines 355-358 (replaced by LIS2DW on EBB36)

Also:
- Fix comments referring to "EBB42" → "EBB36"
- The comment `# Driver4 / # COMMENTED OUT - extruder now on EBB42 CAN board` can be removed entirely once the block is gone

**Total reduction**: ~50 lines.

**Risk**: Low. These are already commented out — they have no functional effect. Git history preserves them if ever needed.

### 5.2 Dead code removal — `macros.cfg`

**Actions**:
1. **Delete `PRIME_LINE` macro** (lines 93-114 — 22 lines). No longer called. LINE_PURGE replaced it on 2026-04-12.
2. **Delete `FANS_ON` macro** (lines 50-52) OR re-implement it properly. The current body is just a commented-out SET_PIN for BentoBox. If BentoBox should auto-start with the print, implement it. If not, delete both the macro AND the `FANS_ON` call in PRINT_START.

**Total reduction**: ~25 lines.

**Risk**: PRIME_LINE removal is zero risk — it's dead code. FANS_ON needs the user's call on intent.

### 5.3 Dead code removal — `ebb_gen2.cfg`

**Action**: delete the commented-out `#[resonance_tester]` at the bottom (3 lines). Moved to printer.cfg on 2026-04-12.

**Total reduction**: 3 lines.

**Risk**: Zero.

### 5.4 Dead code removal — `moonraker.conf`

**Actions**:
1. **Delete the commented-out `[update_manager timelapse]` + `[timelapse]` block** (~13 lines). Timelapse IS installed and symlinked into config/. Check whether it's actually enabled via `[include timelapse.cfg]` in printer.cfg (it's not — add the include if timelapse is desired, OR uninstall moonraker-timelapse if not).
2. **Delete the commented-out `[camera]` / `[mjpeg_stream]` sections** (~10 lines). Crowsnest handles this.

**Total reduction**: ~23 lines.

**Risk**: None to Moonraker. If timelapse is wanted, need to decide: add `[include timelapse.cfg]` to printer.cfg, or uninstall. Right now it's installed but not included → dead install.

### 5.5 Duplicate section definitions

**Action**: Remove the duplicate `[virtual_sdcard]`, `[pause_resume]`, `[display_status]` from printer.cfg (lines 406-409). They're already in `mainsail.cfg`.

**Rationale**: `mainsail.cfg` defines `[virtual_sdcard] path: ~/printer_data/gcodes` — same as printer.cfg. Klipper handles duplicates via last-wins, but having both is redundant and confusing.

**Risk**: Very low. Klipper should continue working identically (mainsail.cfg is included before the duplicates, so mainsail.cfg wins? No wait — `[include mainsail.cfg]` is on line 400 and duplicates are on 406-409, so PRINTER.CFG's versions win currently. Removing them means mainsail.cfg wins, which is identical content. Safe.)

**Verify after**: check Klipper starts without warnings about duplicate sections.

### 5.6 HOTBOX extraction — `printer.cfg` → `chamber.cfg`

**Action**: create a new file `chamber.cfg` and move the HOTBOX cluster from `printer.cfg` to it.

Blocks to move:
- `[heater_generic chamber]` (lines 120-130)
- `[verify_heater chamber]` (lines 132-136)
- `[temperature_fan hotbox]` (lines 138-149)
- `[delayed_gcode HOTBOX_CONTROL]` (lines 154-168)
- `[delayed_gcode HOTBOX_BACKOFF]` (lines 170-173)
- `[output_pin enclosure_fans]` (lines 199-201)
- `[delayed_gcode ENCLOSURE_FAN_CONTROL]` (lines 203-213)

Add `[include chamber.cfg]` to the includes at the bottom of `printer.cfg`.

**Total move**: ~70 lines.

**Risk**: Medium. Move operations are riskier than deletions because one missed section breaks the config. Test thoroughly after: HOTBOX_ON, HOTBOX_OFF, HOTBOX_STATUS should all still work.

### 5.7 Stepper/driver co-location (optional, taste-based)

**Action**: reorder sections in printer.cfg so that each stepper's `[stepper_X]` sits next to its `[tmc5160 stepper_X]` or `[tmc2209 stepper_X]`.

**Current layout**:
```
[stepper_x]    line 20
[stepper_y]    line 34
[stepper_z]    line 47
[stepper_z1]   line 59
[stepper_z2]   line 91
...
[tmc5160 stepper_x]   line 244
[tmc5160 stepper_y]   line 253
[tmc2209 stepper_z]   line 266
[tmc2209 stepper_z1]  line 274
[tmc2209 stepper_z2]  line 287
```

**Proposed layout**: co-locate each stepper with its driver. 210+ lines of vertical distance → 0.

**Risk**: Low (order shuffle only). But medium value — this is taste. If the user prefers the current separation (all steppers grouped, all drivers grouped), skip this.

**Recommendation**: low priority. Do this after the higher-value cleanups if the user wants extra tidiness.

### 5.8 Auto-backup cleanup

**Actions**:
1. Delete all `printer-YYYYMMDD_HHMMSS.cfg` files on the printer. Currently 44 files, ~500 KB.
2. Add the canonical gitignore pattern to the repo's `.gitignore`: `printer-[0-9]*_[0-9]*.cfg`

**Alternative**: Mainsail has a "hide backup files" toggle. Enable it in Mainsail settings. This hides them from the file list without deleting.

**Recommendation**: delete the existing 44 backups (git history is the real record of changes) AND enable the Mainsail hide toggle AND add to gitignore.

**Risk**: Zero. These are duplicates of older `printer.cfg` states, already captured in git history of our repo.

### 5.9 Sonar review

**Action**: decide whether to keep Sonar installed.

`sonar.conf` shows `enable: false`. Sonar is a WiFi keepalive daemon. If the printer has reliable wired/wifi connectivity (which it does — Moonraker has been stable throughout this session), Sonar is unnecessary.

**Options**:
1. Leave installed but disabled (current state) — no harm
2. Uninstall — one less thing to manage, removes the update_manager entry in moonraker.conf

**Recommendation**: if you're never going to use it, uninstall it. If unsure, leave it disabled.

### 5.10 Obico auth_token secret management

**Action**: ensure `moonraker-obico.cfg` is never committed to the git repo.

Currently the file lives only on the printer at `~/printer_data/config/moonraker-obico.cfg` with auth_token `5cd3456a8cf83e29b55a`. The repo's `backups/` folder does NOT currently include this file (the 2026-04-10 snapshot was made BEFORE the file was created at its current path, and the 2024 snapshots have been scrubbed).

If ever doing a new config backup to the repo: explicitly exclude `moonraker-obico.cfg` — or scrub the auth_token before committing. Consider adding to `.gitignore`: `moonraker-obico.cfg`.

**Risk of NOT doing this**: if the token is ever committed and pushed, we'd need another rotation.

---

## 6. Phased migration plan (low-risk ordering)

Execute in this order. **Every phase ends with a FIRMWARE_RESTART verification** to confirm Klipper still boots clean.

### Phase 0 — Prep (pre-migration)

- Pull current config to local repo (we already did this tonight)
- Ensure git working tree is clean so changes can be reviewed
- **No printer must be running for any phase**

### Phase 1 — Pure deletions (safest)

Low-risk dead code removals that can't break anything. Do first to build confidence.

1. Delete `#[extruder]` block from printer.cfg (lines 69-85)
2. Delete `#[filament_switch_sensor material_*]` blocks (lines 87-88, 100-107)
3. Delete `#[fan]` block (lines 191-193)
4. Delete `#[heater_fan heatbreak_cooling_fan]` block (lines 195-197)
5. Delete `#[tmc2209 extruder]` block (lines 282-285)
6. Delete `#[bltouch]` block (lines 328-332)
7. Delete `#[adxl345]` block (lines 355-358)
8. Delete `#[resonance_tester]` from end of ebb_gen2.cfg
9. Fix "EBB42" → "EBB36" in comment text
10. Delete commented-out timelapse + camera blocks from moonraker.conf

**Verify**: FIRMWARE_RESTART. Everything should work identically.

### Phase 2 — Dead macro removal

1. Delete `PRIME_LINE` macro from macros.cfg
2. Decide on FANS_ON: delete OR implement
3. Delete duplicate `[virtual_sdcard]`, `[pause_resume]`, `[display_status]` from printer.cfg (since mainsail.cfg provides them)

**Verify**: FIRMWARE_RESTART. Run a slice-and-print dry run (home + start + cancel) to verify PRINT_START and PRINT_END still work without PRIME_LINE.

### Phase 3 — HOTBOX extraction (highest risk — do LAST)

1. Create `chamber.cfg` with the HOTBOX hardware blocks
2. Remove those same blocks from `printer.cfg`
3. Add `[include chamber.cfg]` to the includes list at the bottom of `printer.cfg`

**Verify**: FIRMWARE_RESTART. Run HOTBOX_ON, HOTBOX_STATUS, HOTBOX_OFF. Verify temperature_fan starts. Verify ENCLOSURE_FAN_CONTROL still triggers on MCU temperature rise. Ideally wait for next ASA print to verify the HOTBOX_CONTROL backoff protection still works.

### Phase 4 — Cleanup accumulated noise

1. Delete all `printer-YYYYMMDD_HHMMSS.cfg` files on the printer
2. Add `printer-[0-9]*_[0-9]*.cfg` to repo `.gitignore`
3. Add `moonraker-obico.cfg` to repo `.gitignore`
4. Enable "hide backup files" in Mainsail settings

**Verify**: Mainsail file view cleaner. No Klipper impact.

### Phase 5 — Optional

1. Stepper/driver co-location (taste-based refactor of printer.cfg section order)
2. Sonar uninstall (if never using)
3. Timelapse decision: add `[include timelapse.cfg]` to printer.cfg OR uninstall moonraker-timelapse

### Rollback

Every phase has an identical rollback: `git checkout printer.cfg ebb_gen2.cfg macros.cfg chamber.cfg moonraker.conf` and FIRMWARE_RESTART. Since we're working via SCP with local git staging, rollback is always one command.

---

## 7. Ongoing maintenance practices

After the cleanup, adopt these conventions:

### 7.1 When adding new config

- If it's user-callable macros → `macros.cfg` (or a purpose-specific macros file if the group grows >10 lines)
- If it's hardware config → the appropriate hardware file (`ebb_gen2.cfg`, `chamber.cfg`, or `printer.cfg` for main-MCU-only hardware)
- If it's a delayed_gcode control loop → co-locate with its hardware (e.g., HOTBOX_CONTROL lives with chamber heater)

### 7.2 When removing hardware

**Delete the config outright. Don't leave it commented.** Git history preserves it. Commented blocks drift out of sync and accumulate.

### 7.3 When running calibration that triggers SAVE_CONFIG

- Let SAVE_CONFIG write to the `#*#` block — that's its job
- Don't manually edit values in the `#*#` block (Klipper will overwrite on next SAVE_CONFIG)
- If you want a "canonical" location for the value (e.g., in a specific include file), copy it into the relevant file's hardware section AND delete from the `#*#` block. Klipper will respect the non-`#*#` definition on next restart.

### 7.4 When updating mainsail.cfg / timelapse.cfg / moonraker_obico_macros.cfg

Don't. These are symlinks to external repos/installs. Edits via the file editor will be lost on next update.

Customisations to Mainsail-style macros (park location, retract amount, etc.) go into your own `printer.cfg` via `_CLIENT_VARIABLE`.

### 7.5 Periodic maintenance

Every ~3 months or whenever you notice clutter:

- Delete accumulated `printer-YYYYMMDD_HHMMSS.cfg` files (they regenerate from SAVE_CONFIG)
- Audit moonraker.conf for dead `[update_manager]` entries (plugins that have been uninstalled)
- Review comments for "TODO" markers or drift
- Diff current state against git — any unexpected changes?

### 7.6 What NOT to do

- **Don't adopt a macro framework** (Klippain, klipper-macros) unless you genuinely want their feature set. They add significant complexity. The current flat-functional pattern is simpler and sufficient for this printer.
- **Don't split `printer.cfg` into per-axis files** (`stepper_x.cfg` etc.). Overkill for 5 steppers.
- **Don't try to move the SAVE_CONFIG `#*#` block out of `printer.cfg`**. Klipper doesn't support it. Years-old open feature request.
- **Don't backup `moonraker-obico.cfg` to the repo without scrubbing the auth token**.

---

## 8. References

All URLs followed and verified during research session 2026-04-12.

### Official Klipper documentation
1. [Klipper Configuration Reference](https://www.klipper3d.org/Config_Reference.html)
2. [Klipper Example Configurations](https://www.klipper3d.org/Example_Configs.html)
3. [Klipper Discourse — Syntax and Parsing Rules](https://klipper.discourse.group/t/klipper-s-printer-cfg-syntax-and-parsing-rules/23425)
4. [LDO Motors — Organizing Klipper Configuration](https://docs.ldomotors.com/en/guides/klipper_multi_cfg_guide)

### Framework configs (Pattern 2 — not adopted but researched)
5. [Frix-x/klippain GitHub](https://github.com/Frix-x/klippain)
6. [Klippain docs/overrides.md](https://github.com/Frix-x/klippain/blob/main/docs/overrides.md)
7. [Klippain user_templates/printer.cfg](https://github.com/Frix-x/klippain/blob/main/user_templates/printer.cfg)
8. [Klippain Issue #534 — SAVE_CONFIG cleanly](https://github.com/Frix-x/klippain/issues/534)
9. [jschuh/klipper-macros GitHub](https://github.com/jschuh/klipper-macros)
10. [klipper-macros README](https://github.com/jschuh/klipper-macros/blob/main/README.md)
11. [rootiest/zippy-klipper_config GitHub](https://github.com/rootiest/zippy-klipper_config)

### Flat functional decomposition configs (Pattern 1 — recommended)
12. [zellneralex/klipper_config GitHub](https://github.com/zellneralex/klipper_config)
13. [garethky/klipper-voron2.4-config GitHub](https://github.com/garethky/klipper-voron2.4-config)

### Official Voron reference
14. [VoronDesign/Voron-2 — Octopus Config](https://github.com/VoronDesign/Voron-2/blob/main/firmware/klipper_configurations/Octopus/Voron2_Octopus_Config.cfg)
15. [Voron Software Configuration Docs](https://docs.vorondesign.com/build/software/configuration.html)

### RatOS framework
16. [RatOS — Includes & Overrides](https://os.ratrig.com/docs/configuration/includes-and-overrides/)

### Mainsail / Fluidd config
17. [mainsail-crew/mainsail-config](https://github.com/mainsail-crew/mainsail-config)
18. [fluidd-core/fluidd-config](https://github.com/fluidd-core/fluidd-config)

### SAVE_CONFIG handling
19. [Klipper Discourse — SAVE_CONFIG in separated file (Feature Request)](https://klipper.discourse.group/t/save-config-in-separated-file/14679)
20. [LDO Motors — Organizing Klipper Configuration](https://docs.ldomotors.com/en/guides/klipper_multi_cfg_guide) (same as Ref 4, different anchor)
21. [Klipper G-Codes reference](https://www.klipper3d.org/G-Codes.html)

### Dead code / comment handling
22. [Voron Forum — Commenting out config code](https://forum.vorondesign.com/threads/is-there-a-easy-way-to-comment-out-a-block-of-config-code.1436/)
23. [Klipper cfg comments — Discourse](https://klipper.discourse.group/t/klipper-cfg-comments/6185)
24. [Klipper Example Configurations — comment pattern](https://www.klipper3d.org/Example_Configs.html) (same as Ref 2, different focus)

### File renaming safety
25. [Klipper Discourse — Unable to open config file errors](https://klipper.discourse.group/t/unable-to-open-config-file-home-pi-printer-data-config-fluidd-cfg/22730)
26. [LDO Organizing Guide — safe rename pattern](https://docs.ldomotors.com/en/guides/klipper_multi_cfg_guide) (same as Ref 4)

### Auto-backup file management
27. [Klipper-Backup FAQ](https://klipperbackup.xyz/faq/)
28. [Klipper-Backup .env.example](https://github.com/Staubgeborener/Klipper-Backup/blob/main/.env.example)
29. [Klipper Discourse — Printer.cfg backups vs moonraker.conf backups](https://klipper.discourse.group/t/printer-cfg-backups-vs-moonraker-conf-backups/7332)
30. [Klipper Discourse — Save backup files to a dedicated directory (Feature Request)](https://klipper.discourse.group/t/save-backup-files-to-a-dedicated-directory/3657)

### Multi-MCU / EBB36 specific
31. [maz0r/klipper_canbus — EBB36 example config](https://github.com/maz0r/klipper_canbus/blob/main/toolhead/example_configs/toolhead_btt_ebbcan_G0B1_v1.1.cfg)
32. [Klipper sample-multi-mcu.cfg](https://github.com/Klipper3d/klipper/blob/master/config/sample-multi-mcu.cfg)
33. [Klipper Discourse — EBB36 v1.2 + Octopus Pro setup](https://klipper.discourse.group/t/setup-ebb36-v1-2-connected-to-octopus-pro/6617)
34. [Klippain overrides.cfg template — multi-MCU guidance](https://github.com/Frix-x/klippain/blob/main/user_templates/overrides.cfg)
35. [maz0r/klipper_canbus toolhead structure guide](https://github.com/maz0r/klipper_canbus/blob/main/toolhead/ebb36-42_v1.1.md)

### Klipper-Backup tool (for ongoing maintenance)
36. [Staubgeborener/Klipper-Backup GitHub](https://github.com/Staubgeborener/Klipper-Backup)
37. [Klipper-Backup Configuration Docs](https://klipperbackup.xyz/configuration/)
38. [Voron Documentation — Backing up to GitHub](https://docs.vorondesign.com/community/howto/EricZimmerman/BackupConfigToGithub.html)

### Documentation practices
39. [Voron Macros Beginner Guide](https://docs.vorondesign.com/community/howto/voidtrance/Klipper_Macros_Beginners_Guide.html)
40. [garethky/klipper-voron2.4-config — per-feature readme pattern](https://github.com/garethky/klipper-voron2.4-config/blob/mainline/printer_data/config/part-cooling.readme.md)

---

*End of plan document. Nothing has been applied to the printer. Awaiting user decision on which phases to execute, and when.*
