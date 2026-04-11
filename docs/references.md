# Best practices reference library

> Curated index of external documentation, guides, tools, and communities for running and maintaining the ZeroG Mercury One.1 described in `../HARDWARE.md`.
> This file points at authoritative sources. It doesn't duplicate them, but it ties each reference to a concrete "when to use" so you're not reading theory in a vacuum.
> Snapshot as of 2026-04-10.

---

## Contents

1. [Start-here: the canonical tuning guides](#start-here-the-canonical-tuning-guides)
2. [Calibration playbook: the recommended sequence](#calibration-playbook)
3. [Core Klipper ecosystem](#core-klipper-ecosystem)
4. [Klippain — the full configuration framework](#klippain--the-full-configuration-framework)
5. [Slicer — OrcaSlicer](#slicer--orcaslicer)
6. [Probe and bed sensing — Beacon](#probe-and-bed-sensing--beacon)
7. [Hardware-specific docs](#hardware-specific-docs)
8. [Firmware flashing](#firmware-flashing)
9. [CAN bus / Katapult / EBB36](#can-bus--katapult--ebb36)
10. [Mechanical tuning: rails, belts, frame squaring](#mechanical-tuning-rails-belts-frame-squaring)
11. [Input shaper: shaper type comparison and workflow](#input-shaper-shaper-type-comparison-and-workflow)
12. [Klipper plugins and tools](#klipper-plugins-and-tools)
13. [Materials: printing-specific tips](#materials-printing-specific-tips)
14. [Print-quality troubleshooting](#print-quality-troubleshooting)
15. [Safety and chamber management](#safety-and-chamber-management)
16. [Video resources: YouTube creators](#video-resources-youtube-creators)
17. [Communities and Discord servers](#communities-and-discord-servers)
18. [MCP servers for 3D printing](#mcp-servers-for-3d-printing)
19. [Local notes and gotchas](#local-notes-and-gotchas)

---

## Start-here: the canonical tuning guides

These are the references to consult first for any print-quality or tuning question. If a quality issue doesn't have a clear answer in one of these three, *then* go wider.

| Reference | Maintained by | Why it's first-line |
|---|---|---|
| **[Ellis' Print Tuning Guide](https://ellis3dp.com/Print-Tuning-Guide/)** ([GitHub](https://github.com/AndrewEllis93/Print-Tuning-Guide)) | Andrew Ellis | The reference. Covers extruder calibration, PA, first layer, retraction, flow, VFAs, troubleshooting. Explicitly Klipper-oriented. Our `TEST_SPEED` macro is his. |
| **[Voron Tuning landing page](https://docs.vorondesign.com/tuning/)** | Voron Design community | Voron-authored tuning section. Use for CoreXY-specific guidance like de-racking that Ellis doesn't cover directly. |
| **[Voron Print Tuning Guide mirror](https://docs.vorondesign.com/community/howto/ellis/print-tuning-guide.html)** | Voron Design community | Same Ellis guide mirrored inside the Voron docs for convenience |
| **[OrcaSlicer Calibration Wiki](https://github.com/OrcaSlicer/OrcaSlicer/wiki/Calibration)** | OrcaSlicer project | Slicer-side calibration wizard. Recommended order is **temperature → flow → pressure advance → max volumetric speed → retraction**. Each has a built-in print in Orca. |

---

## Calibration playbook

The tooling we just installed (Shake&Tune, Klipper TMC Autotune, KAMP) plus what you already have (Beacon, OrcaSlicer) gives you a complete tuning pipeline. Here's the order that makes sense, with the references next to each step.

### Step 0 — Mechanical sanity check (before touching any software)

Don't tune a machine that's mechanically out of whack. Software compensation can paper over problems up to a point, but every hour you spend tuning a misaligned frame is wasted.

1. **Clean and lubricate X/Y/Z linear rails** — [Voron Tuning](https://docs.vorondesign.com/tuning/secondary_printer_tuning.html) section on rail prep. Superlube grease, thin layer, wipe excess.
2. **Check toolhead screws for looseness** — any wobble in the toolhead will poison every downstream measurement.
3. **Verify the frame is square** — see [Mechanical tuning](#mechanical-tuning-rails-belts-frame-squaring) below. If it's not, de-rack the gantry before belt tensioning.
4. **Belt tension: use `COMPARE_BELTS_RESPONSES` to diagnose, Gates app or guitar tuner to adjust** — see [Mechanical tuning](#mechanical-tuning-rails-belts-frame-squaring).
5. **Probe mount rigidity** — Beacon RevH is contactless, but if the duct mounting flexes you'll get inconsistent z_offset between prints.

### Step 1 — Electrical / driver tuning

1. **Stepper current sanity check** — your X/Y TMC5160T Pro at 2.24 A is ≈ 80 % of the LDO 42STH48-2804AC's 2.8 A peak. That's conservative and correct. Z drivers at 1.15 A on Wotrees 17HS4401 (1.5 A rated) is 77 %. Both are fine starting points.
2. **Run Klipper TMC Autotune (when ready)** — add `[autotune_tmc stepper_x]` blocks per motor with the correct `motor:` name from the motor database. See [Klipper plugins](#klipper-plugins-and-tools).

### Step 2 — Heater tuning

1. **Hotend PID** — `PID_CALIBRATE HEATER=extruder TARGET=<print temp>` → `SAVE_CONFIG`. Do this at your typical print temperature for the most common filament.
2. **Bed PID** — same at typical bed temp. Currently saved Kp 21.945 / Ki 1.068 / Kd 112.742 (pre-HotBox era).
3. **Chamber PID** — `PID_CALIBRATE HEATER=chamber TARGET=50`. Current values in `printer.cfg:126-130` (Kp 100 / Ki 0.3 / Kd 30) are initial estimates, not calibrated.

### Step 3 — Resonance and input shaper

This is where Shake&Tune shines. **Run in this order:**

1. **`AXES_MAP_CALIBRATION`** — one-time. Verifies the Beacon's XYZ axes are oriented correctly relative to the printer. Run once per probe install.
2. **`COMPARE_BELTS_RESPONSES`** — drives each CoreXY belt independently via the accelerometer, produces a differential graph of belt frequency response. This is the definitive test for belt tension asymmetry. [Graph interpretation guide](https://github.com/Frix-x/klippain-shaketune/blob/main/docs/macros/compare_belts_responses.md).
3. **`AXES_SHAPER_CALIBRATION`** — Shake&Tune's replacement for `SHAPER_CALIBRATE` / `TEST_RESONANCES`. Better graphs, better recommendations. [Guide](https://github.com/Frix-x/klippain-shaketune/blob/main/docs/macros/axes_shaper_calibrations.md).
4. **(Optional) `CREATE_VIBRATIONS_PROFILE`** — the deep one. Maps resonances across all toolhead directions and speeds, finds the safe speed band for each direction. [Guide](https://github.com/Frix-x/klippain-shaketune/blob/main/docs/macros/create_vibrations_profile.md).

See [Input shaper: shaper type comparison](#input-shaper-shaper-type-comparison-and-workflow) for deciding between MZV / EI / 2HUMP_EI after the calibration runs.

### Step 4 — Per-filament slicer calibration (OrcaSlicer)

Do this **per filament roll**, not per profile. A new roll = repeat these.

1. **Temperature tower** — find the lowest temp that gives good bridging without under-extrusion.
2. **Flow rate** — coarse then fine. Ellis method or Orca's built-in test.
3. **Pressure advance** — after flow rate is stable.
4. **Max volumetric speed** — find the point where surface quality breaks down under sustained extrusion.
5. **Retraction** — usually last, once PA and flow are solid.

References: [Ellis extruder calibration](https://ellis3dp.com/Print-Tuning-Guide/articles/extruder_calibration.html), [OrcaSlicer flow rate guide](https://www.obico.io/blog/orcaslicer-objective-flow-rate-calibration/), [volumetric speed wiki](https://github.com/OrcaSlicer/OrcaSlicer/wiki/volumetric-speed-calib).

### Step 5 — Adaptive features

1. **Enable KAMP adaptive meshing** — uncomment `[include ./KAMP/Adaptive_Meshing.cfg]` in `~/printer_data/config/KAMP_Settings.cfg` and set `enable_object_processing: True` in moonraker.conf's `[file_manager]` section.
2. **Enable Smart Park** — uncomment `[include ./KAMP/Smart_Park.cfg]`. Toolhead parks over the print origin at a safe Z while heating — prevents nozzle ooze on your finished prints.
3. **Enable Line Purge** — uncomment `[include ./KAMP/Line_Purge.cfg]` if you want KAMP's purge in place of your existing `PRIME_LINE` macro (they do similar things).

### Step 6 — Beacon thermal compensation

Once everything above is solid, fine-tune the Beacon Z offset against nozzle temperature drift. See [Probe and bed sensing — Beacon](#probe-and-bed-sensing--beacon).

### Step 7 — Slicer post-processing

Install **Klipper Estimator** for accurate print-time predictions. Currently your gcode time estimates come from Orca's heuristic, which ignores Klipper's actual acceleration profile. Estimator fixes that.

---

## Core Klipper ecosystem

| Reference | URL | When to consult |
|---|---|---|
| Klipper documentation | [klipper3d.org](https://www.klipper3d.org/) | Any config question, pin reference, G-code command reference |
| Klipper config reference | [klipper3d.org/Config_Reference.html](https://www.klipper3d.org/Config_Reference.html) | Every field, every module |
| Klipper G-codes reference | [klipper3d.org/G-Codes.html](https://www.klipper3d.org/G-Codes.html) | Macro authoring, runtime commands |
| Klipper Resonance Compensation | [klipper3d.org/Resonance_Compensation.html](https://www.klipper3d.org/Resonance_Compensation.html) | Shaper type choice, smoothing vs damping trade-offs |
| Klipper Measuring Resonances | [klipper3d.org/Measuring_Resonances.html](https://www.klipper3d.org/Measuring_Resonances.html) | ADXL345 / Beacon / LIS2DW setup, raw `TEST_RESONANCES` |
| Klipper Pressure Advance | [klipper3d.org/Pressure_Advance.html](https://www.klipper3d.org/Pressure_Advance.html) | PA tuning theory and macros |
| Klipper CANBus | [klipper3d.org/CANBUS.html](https://www.klipper3d.org/CANBUS.html) | `can0` config, UUID discovery, interface file |
| Klipper TMC drivers | [klipper3d.org/TMC_Drivers.html](https://www.klipper3d.org/TMC_Drivers.html) | Sensorless homing, stealthChop, coolStep, driver parameter explanations |
| Klipper Config_Checks | [klipper3d.org/Config_Checks.html](https://www.klipper3d.org/Config_Checks.html) | Understanding the `#*#` auto-generated `save_config` block |
| Klipper Discourse forum | [klipper.discourse.group](https://klipper.discourse.group/) | Troubleshooting, community help — more active than GitHub issues |
| Klipper GitHub | [github.com/Klipper3d/klipper](https://github.com/Klipper3d/klipper) | Firmware source, issue tracker |
| Moonraker documentation | [moonraker.readthedocs.io](https://moonraker.readthedocs.io/) | API reference, component config, update manager |
| Moonraker API endpoints | [moonraker.readthedocs.io/en/latest/web_api/](https://moonraker.readthedocs.io/en/latest/web_api/) | Building integrations, automating the printer |
| Mainsail docs | [docs.mainsail.xyz](https://docs.mainsail.xyz/) | Web UI customisation, macros panel, dashboards |
| mainsail-config repo | [github.com/mainsail-crew/mainsail-config](https://github.com/mainsail-crew/mainsail-config) | The `mainsail.cfg` include — standard macro/UI helpers |
| KlipperScreen docs | [klipperscreen.readthedocs.io](https://klipperscreen.readthedocs.io/) | Touchscreen UI, themes, panel customisation |
| Kiauh (installer) | [github.com/dw-0/kiauh](https://github.com/dw-0/kiauh) | Managing Klipper/Moonraker/Mainsail installs, service restarts |
| Danger Klipper (fork docs) | [dangerklipper.io](https://dangerklipper.io/) | A community fork with some experimental features — useful for comparison |

---

## Klippain — the full configuration framework

**[github.com/Frix-x/klippain](https://github.com/Frix-x/klippain)** — "generic Klipper configuration for 3D printers" by Frix-x (the same author who wrote Shake&Tune).

### Why this matters to us

Klippain is a **complete opinionated Klipper config framework**, not a plugin. It provides a pre-built set of macros, startup/shutdown flows, adaptive bed meshing, automated input shaper workflows, and vibration measurement — many of which we'd otherwise have to write ourselves. It's modular: you keep your printer-specific `variables.cfg` / `mcu.cfg` / `overrides.cfg` and let Klippain handle the rest. Updates flow automatically via Moonraker's update manager.

### Supported printers

CoreXY, CoreXZ, Cartesian. Confirmed on Voron V2.4 / Trident / V0 / SwitchWire, TriZero, VzBoT, Ender 5, Ender 3, Prusa — effectively anything with a modern mainboard.

### Documentation

- **[Main README](https://github.com/Frix-x/klippain/blob/main/README.md)** — overview, installation
- **[Configuration guide](https://github.com/Frix-x/klippain/blob/main/docs/configuration.md)** — how to set up `variables.cfg` and `overrides.cfg`
- **[Features list](https://github.com/Frix-x/klippain/blob/main/docs/features.md)** — what you get out of the box
- **[Config reference](https://github.com/Frix-x/klippain/tree/main/config)** — template files

### Why you might want it

- **Adaptive bed mesh** without having to wire up KAMP yourself
- **Automated input shaper workflows** — Klippain integrates natively with Shake&Tune
- **Vibration measurement macros** pre-wired
- **Standard macros** for pause/resume/filament change — consistent with the rest of the Klipper community
- **Auto-update** — Moonraker update_manager tracks Klippain and pulls new features over time

### Why you might not want it

- **Invasive** — it restructures your entire `~/printer_data/config/` tree, which means re-doing all your current customisations (the HotBox loop, the enclosure fan loop, the LED effects wiring) inside Klippain's override format. The migration cost is real.
- **Opinionated** — Klippain's choices may not match yours. You'd be adopting their `PRINT_START` flow instead of your current one.
- **Your current config already works.** Adopting Klippain is a re-platforming, not an incremental upgrade.

**Verdict**: worth knowing about, worth reading the configuration guide for inspiration, but **not a natural next step** unless you specifically want the re-platforming effort. Your current config is fine.

---

## Slicer — OrcaSlicer

| Reference | URL | Notes |
|---|---|---|
| **OrcaSlicer Wiki home** | [github.com/OrcaSlicer/OrcaSlicer/wiki](https://github.com/OrcaSlicer/OrcaSlicer/wiki) | Full wiki — the authoritative source, living with the codebase |
| Calibration landing page | [wiki/Calibration](https://github.com/OrcaSlicer/OrcaSlicer/wiki/Calibration) | Index to every calibration test |
| Flow rate calibration | [wiki/flow-rate-calib](https://github.com/OrcaSlicer/OrcaSlicer/wiki/flow-rate-calib) | Coarse and fine flow rate process |
| Pressure advance | [wiki/pressure-advance-calib](https://github.com/OrcaSlicer/OrcaSlicer/wiki/pressure-advance-calib) | PA tower procedure |
| Adaptive PA calibration | [wiki/adaptive-pressure-advance-calib](https://github.com/OrcaSlicer/OrcaSlicer/wiki/adaptive-pressure-advance-calib) | Klipper-specific — PA varies by acceleration. We don't use this yet |
| Max volumetric speed | [wiki/volumetric-speed-calib](https://github.com/OrcaSlicer/OrcaSlicer/wiki/volumetric-speed-calib) | Relevant because our ASA profile caps at 10 mm³/s while PLA is at 24 — one of these is probably stale |
| Temperature calibration | [wiki/temp-calib](https://github.com/OrcaSlicer/OrcaSlicer/wiki/temp-calib) | Temperature tower test |
| Retraction calibration | [wiki/retraction-calib](https://github.com/OrcaSlicer/OrcaSlicer/wiki/retraction-calib) | Only tune after PA is set |
| Obico OrcaSlicer comprehensive guide | [obico.io blog](https://www.obico.io/blog/orcaslicer-comprehensive-calibration-guide/) | Narrative walkthrough of the above wiki pages |
| OrcaSlicer GitHub | [github.com/OrcaSlicer/OrcaSlicer](https://github.com/OrcaSlicer/OrcaSlicer) | Issue tracker, release notes |

### Orca-specific macros we don't yet use

- **Seam painting** — manual seam override per-object; useful for hiding seams on visible parts
- **Modifier meshes** — per-region settings within a single model (different infill, wall count, etc.)
- **Variable layer height** — lower layer on curves, higher on flat sections. Huge time saver for organic prints.

---

## Probe and bed sensing — Beacon

Our printer uses a Beacon RevH as both Z probe and input-shaper accelerometer. All relevant references:

| Reference | URL | Notes |
|---|---|---|
| Beacon documentation portal | [docs.beacon3d.com](https://docs.beacon3d.com/) | Official vendor docs |
| Beacon quick start | [docs.beacon3d.com/quickstart/](https://docs.beacon3d.com/quickstart/) | Installation, first-calibration flow |
| Beacon commands reference | [docs.beacon3d.com/commands/](https://docs.beacon3d.com/commands/) | `BEACON_CALIBRATE`, `BEACON_AUTO_CALIBRATE`, model management |
| Beacon config reference | [docs.beacon3d.com/config/](https://docs.beacon3d.com/config/) | All parameters — `x_offset`, `y_offset`, `mesh_main_direction`, `mesh_runs` |
| **Beacon Contact mode** | [docs.beacon3d.com/contact/](https://docs.beacon3d.com/contact/) | **High value.** Beacon's Contact mode uses the probe as a load cell — can do automatic Z offset without the paper test, and handles thermal expansion automatically at the start of every print. Requires specific config plus a tuning pass with `BEACON_CALIBRATE_NOZZLE_TEMP_OFFSET`. |
| Beacon Models | [docs.beacon3d.com/models/](https://docs.beacon3d.com/models/) | How to save/restore multiple calibration models (e.g., per build plate) |
| Beacon Klipper plugin | [github.com/beacon3d/beacon_klipper](https://github.com/beacon3d/beacon_klipper) | The Klipper extras `beacon.py` we symlink into `~/klipper/klippy/extras/beacon.py` |
| **[YanceyA/BeaconPrinterTools](https://github.com/YanceyA/BeaconPrinterTools)** | GitHub | **Very high value.** Community macro collection including thermal expansion compensation, contact tuning, quick z-offset, plate switching. Pair with Beacon contact mode. |
| Beacon-KAMP (adaptive mesh) | [github.com/YanceyA/Beacon-KAMP](https://github.com/YanceyA/Beacon-KAMP) | Klipper Adaptive Mesh and Purging integration — mesh only the area your print touches. |
| RevH installation (deep dive) | [deepwiki.com — Plus4-Wiki 2.1](https://deepwiki.com/qidi-community/Plus4-Wiki/2.1-beacon3d-revh-installation-guide) | Third-party RevH install guide — useful for wiring and mounting geometry |
| RatOS Beacon Contact setup | [os.ratrig.com/docs/.../beacon_contact](https://os.ratrig.com/docs/2.1.0rc3/configuration/beacon_contact/) | Another high-quality contact-mode walkthrough |
| Voron3D Wiki: Beacon | [voron3d.wiki/bedleveling/beacon](https://voron3d.wiki/bedleveling/beacon/) | Voron community's collected Beacon knowledge |
| Klipper Discourse: Beacon setup | [klipper.discourse.group — Calibration & Installation of Beacon Probe](https://klipper.discourse.group/t/calibration-installation-of-beacon-probe/21720) | Community troubleshooting thread |

### Beacon-specific tuning discipline (from the docs)

> **Calibrate with a hot bed and mid-range nozzle temp (~150 °C) for bed meshing, then do a final z offset with print temperature nozzle before starting each print.**
>
> This prevents probing damage on thin PEI laminations while keeping the final Z offset accurate for print conditions. Use `contact_max_hotend_temperature` to clamp the safe probing temperature.

### Thermal expansion compensation (the advanced move)

The [BeaconPrinterTools thermal expansion macro](https://github.com/YanceyA/BeaconPrinterTools/blob/main/Tools/Thermal_Expansion_Compensation/Thermal_expansion_compensation.md) uses a nozzle expansion coefficient plus the extruder's target temperature to **automatically apply the correct Z offset at the start of every print** — no manual babystepping per filament type. You run `BEACON_CALIBRATE_NOZZLE_TEMP_OFFSET` once to derive the coefficient, then put the macro call in your `PRINT_START`. Fine-tune first-layer squish with a single variable `beacon_contact_expansion_multiplier` (1.1 = 10 % less squish, 0.9 = 10 % more squish).

**Our current Beacon state**: firmware 2.1.0, RevH hardware, `y_offset: 36` (Tentacool / Goliath Short duct), saved calibration model in `backups/2026-04-10/config/printer.cfg:443-457` (model_temp 25.31 °C, model_offset −0.09 mm). We're **not** yet using Contact mode or thermal expansion compensation.

---

## Hardware-specific docs

### ZeroG Mercury One.1 + Nebula + Hydra + Electronics Enclosure

| Reference | URL | Notes |
|---|---|---|
| **ZeroG documentation root** | [docs.zerog.one](https://docs.zerog.one/) | Main portal — everything ZeroG |
| Mercury One.1 introduction | [docs.zerog.one/manual/build/mercury_eva/introduction](http://docs.zerog.one/manual/build/mercury_eva/introduction) | "Need to know" page |
| Mercury One.1 build instructions | [docs.zerog.one/manual/build/mercury_eva/build_instruction](http://docs.zerog.one/manual/build/mercury_eva/build_instruction) | Step-by-step assembly |
| Mercury One.1 printed files | [docs.zerog.one/manual/build/mercury_eva/printed_files](http://docs.zerog.one/manual/build/mercury_eva/printed_files) | STL downloads, configurator |
| **Mercury One.1 PDF build manual** | [docs.zerog.one/assets/mercury_one_1_instruction_18-02-2024.pdf](https://docs.zerog.one/assets/mercury_one_1_instruction_18-02-2024.pdf) | The single-file canonical build doc (Feb 2024). Too big for WebFetch — download locally if reference needed |
| Hydra bed system | [docs.zerog.one/manual/build/hydra](http://docs.zerog.one/manual/build/hydra) | Bed carrier project (separate from Mercury) |
| Hydra heated bed drawing | [docs.zerog.one/manual/build/hydra/heated_bed_drawing](http://docs.zerog.one/manual/build/hydra/heated_bed_drawing) | DXF drawings for 255 / 275 / 377 / 410 plate variants |
| Electronics Enclosure | [docs.zerog.one/manual/build/electronics_enclosure](http://docs.zerog.one/manual/build/electronics_enclosure) | Bottom-of-printer housing for mainboard, Pi, PSU |
| ZeroG FAQ | [docs.zerog.one/faq](http://docs.zerog.one/faq) | Quick answers |
| ZeroG documentation source | [github.com/ZeroGDesign/documentation](https://github.com/ZeroGDesign/documentation) | Markdown source for the docs — useful when the site is down |
| Nebula Pro frame kit (KB-3D) | [kb-3d.com Nebula Pro](https://kb-3d.com/store/frame-enclosure/3971-zero-g-mercury-one1-nebula-frame-kit-multiple-sizes-colors.html) | Pro is the 255/275 size variant — our size |
| Nebula Pro frame kit (Fabreeko) | [fabreeko.com — Zero G Mercury Nebula Frames by LDO](https://www.fabreeko.com/products/zero-g-mercury-nebula-frames-by-ldo) | Alternate vendor, US |
| ZeroG Discord | Linked from [docs.zerog.one](https://docs.zerog.one/) | Primary build/troubleshooting community |

### VzBot CNC toolhead (Zero G edition)

| Reference | URL | Notes |
|---|---|---|
| VzBot documentation root | [docs.vzbot.org](https://docs.vzbot.org/) | Main VzBot docs |
| Vz-Printhead overview | [docs.vzbot.org/Vz-Printhead](https://docs.vzbot.org/Vz-Printhead) | Printhead introduction |
| **Vz-Printhead CNC GitHub** | [github.com/VzBoT3D/Vz-Printhead-CNC](https://github.com/VzBoT3D/Vz-Printhead-CNC) | CAD files, BOM, README — our toolhead |
| Vz-Printhead printed (printable variant) | [github.com/VzBoT3D/Vz-Printhead-Printed](https://github.com/VzBoT3D/Vz-Printhead-Printed) | Reference for component placement |
| VzBot GitHub org | [github.com/VzBoT3D](https://github.com/VzBoT3D) | All VzBot projects |
| VzBot FAQ | [docs.vzbot.org/faq](https://docs.vzbot.org/faq) | Quick answers — tuning, materials, build tips |

### Hotend — Phaetus Rapido 2 HF

| Reference | URL | Notes |
|---|---|---|
| Phaetus Rapido 2 product page | [phaetus.com/rapido2](https://www.phaetus.com/en-us/products/rapido2) | Vendor official page |
| Rapido 2 HF at 3DJake (EU) | [3djake.com — rapido-hotend-2-hf](https://www.3djake.com/phaetus/rapido-hotend-2-hf) | Full spec sheet from a major EU reseller |
| Rapido 2 HF + PT1000 at West3D | [west3d.com — rapido-hot-end-high-flow-tl-phaetus](https://west3d.com/products/rapido-hot-end-high-flow-tl-phaetus) | Notes on the 104NT vs PT1000 options |
| Rapido Plus 2 HF at MatterHackers | [matterhackers.com — Phaetus Rapido Plus 2](https://www.matterhackers.com/store/l/phaetus-rapido-plus-2-hotend/sk/M3XXFC7Q) | Additional spec reference |

**Our config**: PT1000 with 2200 Ω pullup on EBB36 (`ebb_gen2.cfg:19-21`), max_temp 300 °C, PID-tuned Kp 15.688 / Ki 0.850 / Kd 72.362. The hotend integrates a **90 W planar ceramic heater** — not a cylindrical cartridge. Serviceable thermistor on the underside.

### Nozzles (for future upgrades / abrasive materials)

Our current nozzle is a stock Phaetus hardened steel. If you ever print carbon-fibre PLA, glow-in-the-dark, metal-filled, or wood filament, consider upgrading:

| Option | Spec | When to pick |
|---|---|---|
| **Phaetus Hardened Steel** (current) | Hardened steel, general-purpose abrasive | Carbon fibre, wood, metal-filled, phosphorescent — our current pick |
| **Phaetus DLC Hardened Steel** | DLC-coated hardened steel, better wear | When CF wears the standard hardened steel |
| **Bondtech CHT BiMetal** | Copper body + hardened Vanadium Steel insert. Internal split-core tri-stream for flow | **Best choice for high-flow abrasive.** Higher thermal performance than plain hardened steel |
| **ObXidian (Prusa)** | Hardened steel + corrosion-resistant coating. Same thermal profile as brass | If you want a plug-and-play abrasive nozzle without temperature changes |
| **E3D Revo High Flow** | Revo ecosystem, higher flow than CHT per CNC Kitchen testing | Only if you've already migrated to the Revo ecosystem — not compatible with the Rapido 2 |

References:
- [Bondtech CHT BiMetal landing page](https://www.bondtech.se/product-category/nozzles/bondtech-nozzles/bondtech-cht/cht-for-abrasive-materials/)
- [Phaetus hardened nozzle store](https://www.phaetus.com/en-us/products/hardened-steel-nozzle)
- [CNC Kitchen Revo High Flow review](https://www.cnckitchen.com/blog/e3d-revo-high-flow-review) (useful comparative baseline)

### Extruder — LDO Orbiter 2.5

| Reference | URL |
|---|---|
| Orbiter Projects site | [orbiterprojects.com](https://www.orbiterprojects.com/) |
| LDO Motors Orbiter 2 + 2.5 product | [ldomotors.com](https://www.ldomotors.com/) |
| Orbiter 2 Klipper config examples | [github.com/orbiter3dio/Orbiter-2](https://github.com/orbiter3dio/Orbiter-2) |

**Our config**: `rotation_distance: 4.637`, `full_steps_per_rotation: 200`, run_current 0.85 A, pressure_advance 0.025 default (overridden per-filament).

### Mainboard — BTT Octopus Pro V1.0

| Reference | URL |
|---|---|
| **BTT Octopus Pro V1.0 repo (pinout, schematic, manual)** | [github.com/bigtreetech/BIGTREETECH-OCTOPUS-Pro](https://github.com/bigtreetech/BIGTREETECH-OCTOPUS-Pro) |
| **Voron docs: Octopus Klipper firmware guide** | [docs.vorondesign.com/build/software/octopus_klipper.html](https://docs.vorondesign.com/build/software/octopus_klipper.html) |
| Klipper Octopus Pro sample config | [bigtreetech/BIGTREETECH-OCTOPUS-V1.0/tree/master/Firmware/Klipper](https://github.com/bigtreetech/BIGTREETECH-OCTOPUS-V1.0/tree/master/Firmware/Klipper) |
| Product listing we ordered (UK) | [amazon.co.uk/dp/B09JC2NR1L](https://www.amazon.co.uk/dp/B09JC2NR1L) (STM32F429ZGT6) |

### Toolhead MCU — BTT EBB36 Gen2

| Reference | URL |
|---|---|
| **KB-3D Wiki: BTT EBB36 CAN Toolhead Board** | [wiki.kb-3d.com/en/home/btt/voron/BTT_EBB36](https://wiki.kb-3d.com/en/home/btt/voron/BTT_EBB36) |
| BTT EBB36/42 v1.1 on klipper_canbus docs | [maz0r.github.io/klipper_canbus/toolhead/ebb36-42_v1.1.html](https://maz0r.github.io/klipper_canbus/toolhead/ebb36-42_v1.1.html) |
| BTT EBB36 repo (pinout, schematic) | [github.com/bigtreetech/EBB](https://github.com/bigtreetech/EBB) |

### Stepper drivers

| Reference | URL | Notes |
|---|---|---|
| TMC5160 datasheet (Trinamic) | [analog.com — TMC5160](https://www.analog.com/en/products/tmc5160.html) | For SPI config and register understanding |
| BTT TMC5160T Pro schematic | [github.com/bigtreetech/BIGTREETECH-TMC5160T-PRO-V1.0](https://github.com/bigtreetech/BIGTREETECH-TMC5160T-PRO-V1.0) | Schematic, dimensions, jumper settings |
| TMC2209 datasheet | [analog.com — TMC2209](https://www.analog.com/en/products/tmc2209.html) | Same for the Z and extruder drivers |
| Klipper TMC drivers doc | [klipper3d.org/TMC_Drivers.html](https://www.klipper3d.org/TMC_Drivers.html) | Sensorless homing, stealthChop, coolStep, driver parameter explanations |

### Steppers

| Reference | URL | Notes |
|---|---|---|
| LDO 42STH48-2804AC | [ldomotors.com Super Power Stepper Motor](https://www.ldomotors.com/) | 2.8 A peak, 0.55 Nm — our X/Y motors |
| Wotrees 17HS4401 | Search Amazon / AliExpress | 1.5 A, 0.42 Nm — our Z motors |

---

## Firmware flashing

Guides for flashing Klipper firmware to each MCU on the printer. Relevant when upgrading Klipper, recovering from corruption, or changing compile-time config.

### Octopus Pro V1.0 (STM32F429ZGT6)

The Octopus Pro has a 32 KiB bootloader already flashed from the factory. You can flash via **DFU mode** (USB-only, no programmer needed) or via **SD card** (the classic BTT method).

**`make menuconfig` settings for our board**:
- Enable extra low-level configuration options
- Micro-controller Architecture: STMicroelectronics STM32
- Processor model: **STM32F429**
- Bootloader offset: **32KiB bootloader**
- Clock Reference: **8 MHz crystal**
- Communication interface: **USB (on PA11/PA12)**

**Flash via DFU**:
1. Hold BOOT0 while tapping RESET on the Octopus
2. `lsusb` on the Pi → find the STM Device in DFU mode (typically `0483:df11`)
3. `cd ~/klipper && make flash FLASH_DEVICE=0483:df11`
4. Verify with `ls /dev/serial/by-id`

**References**:
- [Voron docs: Octopus Klipper firmware](https://docs.vorondesign.com/build/software/octopus_klipper.html) — canonical
- [BIGTREETECH-OCTOPUS-V1.0 Klipper README](https://github.com/bigtreetech/BIGTREETECH-OCTOPUS-V1.0/blob/master/Firmware/Klipper/README.md) — vendor
- [Sanket's journal: flashing BTT Octopus Pro using Pi](https://www.sanketsjournal.com/articles/20240303-flashing-btt-octopus-pro-with-klipper-using-pi) — third-party walkthrough
- [Klipper Discourse: `make flash` & Octopus Pro](https://klipper.discourse.group/t/make-flash-octopus-pro-board/11953) — troubleshooting

### EBB36 Gen2 (STM32G0B1)

The EBB36 has **Katapult** flashed as its bootloader (we installed it during the CAN-bus preparation). Katapult supports USB, UART, and CAN flashing.

**`make menuconfig` for EBB36 over USB**:
- Architecture: STM32
- Processor model: STM32G0B1
- Bootloader offset: **8 KiB bootloader** (Katapult footprint)
- Clock Reference: **8 MHz crystal**
- Communication interface: **USB (on PA11/PA12)** or **CAN bus** (on PB0/PB1) depending on how you're connecting

**Flash via Katapult**:
```
cd ~/katapult/scripts
python3 flashtool.py -d /dev/serial/by-id/usb-Katapult_stm32g0b1xx_XXX -f ~/klipper/out/klipper.bin
```

**References**:
- [Katapult GitHub](https://github.com/Arksine/katapult) — bootloader source + scripts
- [KB-3D Wiki: BTT EBB36](https://wiki.kb-3d.com/en/home/btt/voron/BTT_EBB36) — complete first-flash walkthrough
- [maz0r klipper_canbus EBB36/42 v1.1](https://maz0r.github.io/klipper_canbus/toolhead/ebb36-42_v1.1.html) — details with CAN

### Beacon RevH

Beacon firmware is updated via a script rather than raw DFU:
```
cd ~/beacon_klipper
python3 update_firmware.py
```

**References**:
- [Beacon commands: firmware update](https://docs.beacon3d.com/commands/) — vendor docs
- [github.com/beacon3d/beacon_klipper](https://github.com/beacon3d/beacon_klipper) — source + update_firmware.py

---

## CAN bus / Katapult / EBB36

Relevant to the half-finished CAN conversion on this printer.

| Reference | URL | Notes |
|---|---|---|
| **Katapult (Arksine)** | [github.com/Arksine/katapult](https://github.com/Arksine/katapult) | The CAN bus (and USB, UART) bootloader we have flashed to the EBB36 |
| VoronTools EBB CAN guide | [github.com/EricZimmerman/VoronTools/blob/main/EBB_CAN.md](https://github.com/EricZimmerman/VoronTools/blob/main/EBB_CAN.md) | Step-by-step Katapult → Klipper flow |
| FracktalWorks CAN setup | [github.com/FracktalWorks/CANBUS-setup-documentation](https://github.com/FracktalWorks/CANBUS-setup-documentation) | Alternative community walkthrough |
| Klipper CANBus Connections doc | [klipper3d.org/CANBUS.html](https://www.klipper3d.org/CANBUS.html) | Official Klipper guide — hostside `can0` interface configuration, systemd network file, UUID discovery |
| Klipper Discourse: Katapult step-by-step | [klipper.discourse.group — step by step guide for katapult](https://klipper.discourse.group/t/is-there-a-step-by-step-guide-for-katapult/14973) | Community thread with the fewest gotchas |
| E3NG CANbus install walkthrough | [e3nglog.themakermedic.com/software_install-CANBUS.html](https://e3nglog.themakermedic.com/software_install-CANBUS.html) | Uses SKR Mini E3 as USB↔CAN bridge; our Octopus Pro v1.0 can bridge natively |

**Current state**: Katapult is installed on the EBB36, but `can0` is not configured on the Pi — the EBB is currently running over USB via the main umbilical. See `HARDWARE.md` → *Known gaps* for context.

---

## Mechanical tuning: rails, belts, frame squaring

The step-0 discipline from the calibration playbook. Software can't fix mechanical problems.

### Frame squaring and gantry de-racking

| Reference | URL | Notes |
|---|---|---|
| **Voron gantry racking video (Nero 3D)** | Linked from [docs.vorondesign.com/tuning/secondary_printer_tuning.html](https://docs.vorondesign.com/tuning/secondary_printer_tuning.html) | The canonical de-racking video for CoreXY gantries |
| Ellis first layer troubleshooting | [ellis3dp.com/Print-Tuning-Guide/articles/troubleshooting/first_layer_squish_consistency_issues](https://ellis3dp.com/Print-Tuning-Guide/articles/troubleshooting/first_layer_squish_consistency_issues/first_layer_inconsistency.html) | First-layer inconsistency maps to specific mechanical causes |
| Ellis VFAs (Vertical Fine Artifacts) | [ellis3dp.com/.../vfas.html](https://ellis3dp.com/Print-Tuning-Guide/articles/troubleshooting/vfas.html) | Vertical artefacts are *almost always* mechanical, not slicer |

**Procedure summary** (paraphrased from the Voron guide):
1. Power off the motors so the gantry moves freely.
2. Push the gantry to the back of the machine — both sides hit the rear plate simultaneously.
3. If one side leads the other, your gantry is racked: loosen the relevant belt, square the gantry physically, re-tighten.
4. Do this *before* setting belt tension.

### Belt tension

Two complementary methods: **frequency measurement** (objective, quantitative) and **Shake&Tune's `COMPARE_BELTS_RESPONSES`** (relative, CoreXY-specific).

| Reference | URL | What it's for |
|---|---|---|
| **Mark Rehorst: CoreXY layout and belt tensioning** | [drmrehorst.blogspot.com — CoreXY Mechanism Layout](https://drmrehorst.blogspot.com/2018/08/corexy-mechanism-layout-and-belt.html) | The canonical theory piece. "The goal is getting X and Y square — not matching absolute tensions." |
| **Gates Carbon Drive app** | [gatescarbondrive.com — Handling & Tension](https://www.gatescarbondrive.com/resources/handling-and-tension) | Android/iOS app that measures belt frequency by plucking it like a guitar string. Typical target for CoreXY is 50–60 Hz, but *achieve squareness first, frequency is secondary*. |
| **3D Distributed: Belt frequency and tensioning** | [3ddistributed.com/belt-frequency-and-tensioning](https://3ddistributed.com/belt-frequency-and-tensioning/) | Practical frequency-method guide |
| Klipper Discourse: precise belt tension for CoreXY | [klipper.discourse.group — Precise belt tension for CoreXY](https://klipper.discourse.group/t/precise-belt-tension-for-corexy/367) | Community discussion of numerical targets |
| Shake&Tune `COMPARE_BELTS_RESPONSES` docs | [github.com/Frix-x/klippain-shaketune/.../compare_belts_responses.md](https://github.com/Frix-x/klippain-shaketune/blob/main/docs/macros/compare_belts_responses.md) | The CoreXY-specific differential belt diagnostic |
| RatRig belt tension matters (community discussion) | [forum.3dprintbeginner.com/t/belt-tension-matters](https://forum.3dprintbeginner.com/t/belt-tension-matters/45) | Good real-world examples |

### Rail care and maintenance

| Reference | URL |
|---|---|
| Voron tuning: secondary printer tuning | [docs.vorondesign.com/tuning/secondary_printer_tuning.html](https://docs.vorondesign.com/tuning/secondary_printer_tuning.html) |
| Voron materials selection (for printed parts) | [docs.vorondesign.com/materials.html](https://docs.vorondesign.com/materials.html) |

**Standard recommendation**: Superlube synthetic grease, thin layer, wiped in. Re-lube every ~500 hours or when you hear dry-rail sounds.

---

## Input shaper: shaper type comparison and workflow

Our current input shaper uses MZV at X=63.8 Hz / Y=43.4 Hz. Whether that's optimal depends on whether we have multiple resonance peaks or just one dominant peak per axis. Shake&Tune will tell us.

### Shaper types ordered by smoothing

From the [Klipper Resonance Compensation docs](https://www.klipper3d.org/Resonance_Compensation.html):

**ZV < MZV < ZVD ≈ EI < 2HUMP_EI < 3HUMP_EI**

Lower on the list = more vibration cancellation, more corner smoothing.

### Which one to pick?

| Shaper | Best for | Trade-off |
|---|---|---|
| **ZV** | Very low ringing frequencies (~25 Hz and below) | Very sensitive to frequency measurement errors. Rarely used on CoreXY. |
| **MZV** | Single dominant resonance peak. Standard CoreXY starting choice. Our current setting. | Minimal smoothing but less forgiving of frequency drift. |
| **EI** | Printers with multiple resonance peaks or where the frequency varies with temperature / wear | Slightly more smoothing than MZV, more robust |
| **2HUMP_EI** | Specifically suppressing two distinct resonance peaks | More smoothing; configure `shaper_freq` at the lower peak |
| **3HUMP_EI** | Three-peak cases (bed slingers or highly flexible toolheads) | Heaviest smoothing. Rarely needed on a CoreXY. |

### Decision rule

Run `AXES_SHAPER_CALIBRATION` (Shake&Tune). Look at the generated graph:

- **Single sharp peak** → MZV is fine. Use the frequency of that peak.
- **Multiple peaks within a few Hz** → EI, with `shaper_freq` at the midpoint.
- **Two peaks separated by >5 Hz** → 2HUMP_EI, with `shaper_freq` at the lower peak.
- **Very low peak (< 25 Hz)** → Mechanical problem first — toolhead too heavy, frame too flexible, or input shaping won't save you.

### Additional reading

| Reference | URL |
|---|---|
| **Klipper Resonance Compensation** (official) | [klipper3d.org/Resonance_Compensation.html](https://www.klipper3d.org/Resonance_Compensation.html) |
| Klipper Measuring Resonances (official) | [klipper3d.org/Measuring_Resonances.html](https://www.klipper3d.org/Measuring_Resonances.html) |
| DeepWiki: Input Shaping and Resonance Compensation | [deepwiki.com/Klipper3d/klipper/8.2](https://deepwiki.com/Klipper3d/klipper/8.2-input-shaping-and-resonance-compensation) |
| ThinkRobotics input shaper tuning guide | [thinkrobotics.com — Input Shaper Tuning](https://thinkrobotics.com/blogs/learn/input-shaper-tuning-with-klipper-complete-calibration-guide) |
| Shake&Tune `AXES_SHAPER_CALIBRATION` docs | [github.com/Frix-x/klippain-shaketune/.../axes_shaper_calibrations.md](https://github.com/Frix-x/klippain-shaketune/blob/main/docs/macros/axes_shaper_calibrations.md) |

---

## Klipper plugins and tools

### Installed and active

| Plugin | URL | Role |
|---|---|---|
| **klipper-led_effect** (julianschill) | [github.com/julianschill/klipper-led_effect](https://github.com/julianschill/klipper-led_effect) | LED effects engine — drives our 29-element neopixel chain via `led_marcos.cfg` [sic] |
| **beacon_klipper** (beacon3d) | [github.com/beacon3d/beacon_klipper](https://github.com/beacon3d/beacon_klipper) | Beacon probe driver — dev channel |
| **klippain-shaketune** (Frix-x) | [github.com/Frix-x/klippain-shaketune](https://github.com/Frix-x/klippain-shaketune) | **Just installed.** CoreXY belt comparison, improved shaper tuning, vibration profiles |
| **klipper_tmc_autotune** (andrewmcgr) | [github.com/andrewmcgr/klipper_tmc_autotune](https://github.com/andrewmcgr/klipper_tmc_autotune) | **Just installed, dormant.** Per-motor TMC driver auto-tune. Needs `[autotune_tmc stepper_x]` blocks to activate |
| **Klipper-Adaptive-Meshing-Purging** (kyleisah) | [github.com/kyleisah/Klipper-Adaptive-Meshing-Purging](https://github.com/kyleisah/Klipper-Adaptive-Meshing-Purging) | **Just installed, dormant.** All sub-includes commented out in `KAMP_Settings.cfg` |
| **moonraker-obico** | [github.com/TheSpaghettiDetective/moonraker-obico](https://github.com/TheSpaghettiDetective/moonraker-obico) | Remote monitoring, AI print-failure detection |
| **moonraker-timelapse** | [github.com/mainsail-crew/moonraker-timelapse](https://github.com/mainsail-crew/moonraker-timelapse) | Timelapse generation — half-wired (symlinked config, `[update_manager]` entry commented out) |
| **crowsnest** | [github.com/mainsail-crew/crowsnest](https://github.com/mainsail-crew/crowsnest) | Webcam stack — ustreamer + camera-streamer orchestration |
| **sonar** | [github.com/mainsail-crew/sonar](https://github.com/mainsail-crew/sonar) | Wi-Fi keep-alive — enabled but currently not running |

### Strongly recommended but not yet installed

| Tool | URL | Why |
|---|---|---|
| **Klipper Estimator** (Annex-Engineering) | [github.com/Annex-Engineering/klipper_estimator](https://github.com/Annex-Engineering/klipper_estimator) | Post-processes slicer G-code and replaces the optimistic estimated-time with one that uses Klipper's actual acceleration model. Accuracy improves from "rough" to **±5 seconds**. Works as an Orca post-processor, writes the new time back into `M73` and the gcode metadata that Mainsail reads. Install guide: [freakyDude's blog](https://blog.freakydu.de/posts/2025-09-05-improved_klipper_print_estimation/). **Biggest daily-quality-of-life upgrade of anything listed here.** |
| **Klipper-Backup** (Staubgeborener) | [github.com/Staubgeborener/Klipper-Backup](https://github.com/Staubgeborener/Klipper-Backup) · [klipperbackup.xyz](https://klipperbackup.xyz/) | A lightweight script that auto-commits your `printer_data/config/` to GitHub on a schedule. **We're doing this manually via rsync into this repo; Klipper-Backup would automate it from the printer side, pushing directly.** Alternative to our current workflow. |
| **YanceyA/BeaconPrinterTools** | [github.com/YanceyA/BeaconPrinterTools](https://github.com/YanceyA/BeaconPrinterTools) | Macro collection for Beacon contact mode, thermal expansion compensation, quick z-offset, plate switching. Pair with Beacon's Contact feature. |
| **Beacon-KAMP** (YanceyA) | [github.com/YanceyA/Beacon-KAMP](https://github.com/YanceyA/Beacon-KAMP) | KAMP integration tailored for Beacon — uses contact mode for homing, adaptive mesh calibration. Only relevant after Contact mode is enabled. |

### Worth knowing about (not installing today)

| Tool | URL | Why not now |
|---|---|---|
| **Klippain** (Frix-x) | [github.com/Frix-x/klippain](https://github.com/Frix-x/klippain) | Full config framework. Adopting it is a re-platforming effort. See the dedicated [Klippain](#klippain--the-full-configuration-framework) section. |
| Fluidd (alternative web UI) | [docs.fluidd.xyz](https://docs.fluidd.xyz/) | Alternative to Mainsail. You're happy with Mainsail; no reason to switch. |
| Beacon-TAP (YanceyA) | [github.com/YanceyA/BeaconPrinterTools](https://github.com/YanceyA/BeaconPrinterTools) | Beacon's equivalent of Voron TAP. Our toolhead is not TAP-compatible. |

---

## Materials: printing-specific tips

### Voron materials guidance

The [Voron materials documentation](https://docs.vorondesign.com/materials.html) is the canonical reference for what filaments play well with enclosed CoreXY printers. Summary for our use case:

### ABS / ASA (chamber needed)

Our HotBox chamber heater maxes at 70 °C (software limit). The Voron docs point out:

- **Typical chamber temp for ABS/ASA**: 55–60 °C passively, achievable with door closed and bed at 110 °C without active chamber heat
- **Hard ceiling**: chamber > 80 °C starts causing Voron-standard printed parts to warp, belts to soften, and bearing grease to thin. **Do not exceed.** Our 70 °C soft limit and 87 °C hardware backoff are both safely below this.
- **Bed temp**: ABS/ASA glass transition is 105 °C, standard bed target is 100–110 °C. Experience suggests 95 °C works with good adhesion aids and avoids warping issues
- **Fan policy**: don't be afraid of part cooling inside a heated chamber. ABS/ASA still need some fan (20–35 %) — zero fan isn't correct even for ABS
- **Humidity**: ASA is hygroscopic. Dry it. Your Sunlu filament dryer is the right tool.
- **Surface prep**: clean PEI with soap and warm water before printing. Skin oil is the silent killer of adhesion.
- **Heat soak**: let the printer reach steady-state chamber temperature before starting the print. For us with no insulation, that's the ~45 minute soak you mentioned.

References:
- [Voron materials](https://docs.vorondesign.com/materials.html)
- [Voron forum — chamber temp for tall ASA](https://forum.vorondesign.com/threads/chamber-temp-for-tall-single-wall-asa.2291/)
- [Voron forum — ABS experimenting](https://forum.vorondesign.com/threads/abs-experimenting.1759/)
- [Standard Print Co ASA quick-start](https://standardprintco.com/read/asa/quick-start-guide)

### PLA (no chamber needed, may actively hurt)

- Chamber temperature **should not exceed ~35 °C** for PLA. The HotBox should be **off** for PLA prints.
- Our enclosure fans are driven by Pi/MCU temp, not chamber temp — they won't help cool the chamber for PLA. Leave the door open for long PLA prints in a warm room.
- Bed 60 °C, nozzle 200–220 °C for most PLA. Our `eSUN PLA Plus` base profile uses 220 °C.

### PETG

- Moderate chamber (30–40 °C is plenty). Bed 70–80 °C. Nozzle 230–245 °C.
- Very prone to stringing — pressure advance and retraction tuning matter more than with other materials.
- Sticks very aggressively to PEI — use a release agent (glue stick) or accept that you'll damage the PEI surface on removal.

### Flexible / TPU

- Not ideal for a direct drive with a relatively long filament path like ours. Reduce retraction. Print slow (30 mm/s or less). Disable part cooling or use very low fan speed.
- Your extruder gear ratio of 7.5:1 on Orbiter 2.5 is actually good for TPU — the high ratio helps with soft filament control.

### Carbon / glass-fibre composites

- **Hardened nozzle required.** You have one. Good.
- Dry the filament. CF filaments are very hygroscopic.
- 0.4 mm nozzle minimum — 0.6 mm preferred. Fibres clog 0.2–0.3 mm nozzles fast.
- Expect to replace hardened steel nozzles every 200–400 hours of CF printing.

---

## Print-quality troubleshooting

When a print comes off and something's wrong, this is where to look.

### Primary visual-defect references

| Reference | URL | Strength |
|---|---|---|
| **Ellis troubleshooting index** | [ellis3dp.com/Print-Tuning-Guide/articles/troubleshooting](https://ellis3dp.com/Print-Tuning-Guide/) | Klipper-specific, deep, honest about mechanical vs slicer causes |
| **Simplify3D Print Quality Troubleshooting** | [simplify3d.com/resources/print-quality-troubleshooting/](https://www.simplify3d.com/resources/print-quality-troubleshooting/) | Classic illustrated guide — generic slicer but photos are canonical |
| **3DXTech 27 FDM problems** | [3dxtech.com — 27 Common FDM 3D Printing Problems](https://www.3dxtech.com/blogs/trouble-shooting/27-common-fdm-3d-printing-problems-and-how-to-fix-them) | Material-specific troubleshooting (they sell engineering filaments) |
| **Hydra Research troubleshooting** | [hydraresearch3d.com/print-quality-troubleshooting](https://www.hydraresearch3d.com/print-quality-troubleshooting) | Professional-tier reference |
| **Printing Atoms FDM troubleshooting** | [printingatoms.com/3d-printing-troubleshooting-issues](https://printingatoms.com/3d-printing-troubleshooting-issues/) | Good photo library |

### Defect → likely cause cheat sheet

| What you see | Most likely cause (check in order) |
|---|---|
| **Stringing / oozing / webbing** | 1. Wet filament (dry it), 2. Retraction too low, 3. Nozzle too hot, 4. Travel speed too slow over gaps |
| **Ghosting / ringing / echoing** | 1. Input shaper not tuned, 2. Belt tension asymmetry (run `COMPARE_BELTS_RESPONSES`), 3. Print speed too high, 4. Toolhead loose, 5. Frame not on a stable surface |
| **Warping** | 1. Bed adhesion (dirty plate, wrong temp, no brim), 2. Chamber too cold for ABS/ASA, 3. Part cooling too aggressive on layer 1 |
| **Under-extrusion** | 1. Flow rate not calibrated, 2. Nozzle partially clogged, 3. Extruder gear slipping (tension), 4. Temperature too low, 5. Filament diameter wrong |
| **Over-extrusion / blobs** | 1. Flow rate too high, 2. Pressure advance too low (leading blobs), 3. Temperature too high |
| **Layer shifts** | 1. Belt tension, 2. Step skipping from TMC current too low, 3. Acceleration too high for current, 4. Mechanical obstruction |
| **First layer inconsistency** | 1. Bed not meshed or mesh is stale, 2. Z offset drifted (babystep and `Z_OFFSET_APPLY_PROBE`), 3. PEI surface dirty, 4. Frame not square |
| **Z banding** | 1. Z rod wobble (Oldham couplers should fix this — ours does), 2. Z motor vibration, 3. TMC driver stealthChop on Z, 4. Temperature oscillation |
| **Corner bulges** | 1. Pressure advance too low, 2. Corner velocity too high, 3. Under-extrusion on straight sections (so PA correction over-extrudes in corners) |
| **VFAs (vertical fine artifacts)** | Almost always mechanical. See [Ellis VFAs](https://ellis3dp.com/Print-Tuning-Guide/articles/troubleshooting/vfas.html). Not slicer. |

---

## Safety and chamber management

Because this printer has a heated enclosure (HotBox up to 70 °C), a 600 W AC bed heater on an SSR, no door interlock, and no exhaust, safety discipline matters.

| Topic | Reference |
|---|---|
| Thermal runaway in Klipper | [klipper3d.org/Config_Reference.html#verify_heater](https://www.klipper3d.org/Config_Reference.html#verify_heater) — we use `verify_heater chamber` with `max_error 600, check_gain_time 300, hysteresis 5, heating_gain 0.5` |
| Snapmaker — 3D printer fire safety | [snapmaker.com/blog — fire-safety-causes-prevention](https://www.snapmaker.com/blog/3d-printer-fire-safety-causes-prevention-best-practices/) |
| Alveo3D — 3D printer fires | [alveo3d.com/en/3d-printer-fire-safety](https://www.alveo3d.com/en/3d-printer-fire-safety/) |
| JLC3DP — fire safety risks and protection | [jlc3dp.com/blog/3d-printer-fire-safety](https://jlc3dp.com/blog/3d-printer-fire-safety) |
| Qidi Tech — thermal runaway prevention | [qidi3d.com/blogs/news/prevent-thermal-runaway-3d-printer](https://qidi3d.com/blogs/news/prevent-thermal-runaway-3d-printer) |
| 3DPUT — heated enclosures | [3dput.com/heated-3d-printer-enclosures](https://3dput.com/heated-3d-printer-enclosures/) |

### Safety checklist

- [ ] Klipper `[verify_heater]` enabled on every heater (currently: hotend via default, bed via default, chamber explicit)
- [ ] Physical thermal fuse on the HotBox element (**yes** — 125 °C ceramic, `github.com/nark3d/HotBox` BOM)
- [ ] Smoke/fire alarm in the room (**operational — confirm**)
- [ ] Non-flammable surface under printer (**operational — confirm**)
- [ ] VOC ventilation for ASA/ABS (**no — TBD**; currently no exhaust fan and no door interlock — long ASA prints should only run with supervision or a windowed room)
- [ ] SSR heat dissipation — Heschen SSR-40DA driving 600 W @ 240 V ≈ 2.5 A is comfortably inside spec but benefits from a finned heatsink (**TBD — confirm installed**)
- [ ] Mains-side fused protection upstream of the SSR (**TBD — confirm**)
- [ ] Physical kill switch (**yes** — BTT Relay V1.2 with 10 A rocker)

---

## Video resources: YouTube creators

Reading is fine, but some topics (mechanical de-racking, belt tension feel, toolhead assembly) are much faster to learn by watching.

| Creator | URL | Focus |
|---|---|---|
| **Nero 3D (Taylor Benoit)** | [@3dpNero on YouTube](https://www.youtube.com/@3dpNero) · [Twitter](https://x.com/3dpNero) | The go-to Voron channel. Huge library of build livestreams, tuning walkthroughs, product reviews. His **gantry racking video** is directly cited in the Voron tuning docs. |
| **Voron Video Guides** | [docs.vorondesign.com/community/video_guides.html](https://docs.vorondesign.com/community/video_guides.html) | Voron's curated list of community video guides. Start here for any Voron-shape CoreXY question. |
| **Steamengine 3D** | Search YouTube | Klipper deep dives, specifically around RatRig and CoreXY tuning |
| **Gabriel Boulanger** | Search YouTube | Voron 0 builds, Klipper macros |
| **CNC Kitchen (Stefan)** | [youtube.com/@CNCKitchen](https://www.youtube.com/@CNCKitchen) | Systematic testing of nozzles, materials, parameters. The definitive "does this actually work" channel. |
| **Thomas Sanladerer** | [youtube.com/@MadeWithLayers](https://www.youtube.com/@MadeWithLayers) | Long-form reviews and explainers. Broader than Klipper-specific but excellent fundamentals. |

### Specific Nero 3D videos worth finding

- "Voron Gantry Racking" — the one the Voron docs link to
- Revo hotend comparisons (for general understanding even though we use Phaetus)
- Shake&Tune / Klippain walkthroughs

---

## Communities and Discord servers

| Community | Link | Focus |
|---|---|---|
| **ZeroG Discord** | Invite from [docs.zerog.one](https://docs.zerog.one/) | Mercury One.1, Nebula, Hydra, Electronics Enclosure — the primary build support community |
| **VzBot Discord** | Invite from [docs.vzbot.org](https://docs.vzbot.org/) | VzBot printers and VzBot printheads (our toolhead) |
| **Voron Discord** | Invite from [vorondesign.com](https://vorondesign.com/) | General CoreXY / Klipper expertise — even for non-Voron printers |
| **Beacon Discord** | Linked from [docs.beacon3d.com](https://docs.beacon3d.com/) | Beacon-specific support |
| **Klippain Discord** | Invite from [github.com/Frix-x/klippain](https://github.com/Frix-x/klippain) | Same author as Shake&Tune — tuning and framework support |
| Klipper Discourse | [klipper.discourse.group](https://klipper.discourse.group/) | Official Klipper forum — best for deep troubleshooting |
| r/klippers | [reddit.com/r/klippers](https://www.reddit.com/r/klippers/) | General Klipper community |
| r/voroncorexy | [reddit.com/r/voroncorexy](https://www.reddit.com/r/voroncorexy/) | CoreXY community |
| r/3Dprinting | [reddit.com/r/3Dprinting](https://www.reddit.com/r/3Dprinting/) | Everything 3D printing |

---

## MCP servers for 3D printing

These are Model Context Protocol servers that let AI assistants (Claude, etc.) talk directly to printers or adjacent tooling. **We installed the first one in this pass.**

| Server | Scope | Maturity | Notes |
|---|---|---|---|
| **DMontgomery40/mcp-3D-printer-server** ⭐ **INSTALLED** | Multi-protocol: Moonraker, OctoPrint, Duet, Bambu, Prusa Connect, Creality Cloud, Repetier | **173 stars**, GPL v2, npm-installable (`npm install -g mcp-3d-printer-server`), Docker support, TypeScript 4.9+, Node 18+ | [GitHub](https://github.com/DMontgomery40/mcp-3D-printer-server) · [npm](https://www.npmjs.com/package/mcp-3d-printer-server) · [Awesome MCP listing](https://mcpservers.org/servers/DMontgomery40/mcp-3D-printer-server). Currently configured in `.mcp.json` pointing at `192.168.x.x:7125`. Activates on next Claude Code session. |
| Charleslotto/klipper-mcp | Klipper via Moonraker | Early / VS Code-oriented | [Glama listing](https://glama.ai/mcp/servers/@Charleslotto/klipper-mcp) · [LobeHub](https://lobehub.com/mcp/charleslotto-klipper-mcp) |
| grego33/klipper-config-mcp | Klipper config editing | Early | [GitHub](https://github.com/grego33/klipper-config-mcp) |
| OctoEverywhere MCP | Status/control via OctoEverywhere cloud | Vendor | [Blog post](https://blog.octoeverywhere.com/mcp-server-for-3d-printing/) — requires OctoEverywhere account |

### Capabilities now available through the MCP (after Claude Code restart)

Once `.mcp.json` loads on the next session, Claude Code can:
- Read printer status: temperatures, toolhead position, print progress, current filename
- Stream `klippy.log` without re-rsyncing
- List, upload, and delete files in `~/printer_data/gcodes/`
- Send gcode commands (including `COMPARE_BELTS_RESPONSES`, `FIRMWARE_RESTART`, etc.)
- Query bed mesh data
- Read/write temperature targets
- Emergency stop
- STL manipulation (scale, rotate, section) via bundled helpers
- Slice locally via bundled slicer path (if configured)

Security boundary is entirely **Moonraker's `[authorization]` trusted_clients list**. Our subnet (192.168.0.0/16) is trusted, so no API key is required but also no extra authorization gate exists beyond that. Treat Claude sends to the printer with appropriate caution.

---

## Local notes and gotchas

Things specific to this machine that you'd otherwise forget:

### Klipper repo is "dirty"

Klippy.log shows the Klipper version as `v0.13.0-540-g57c2e0c96-dirty`. This is **not a problem** — the "dirty" flag is caused by **untracked symlinks** in the working tree:

- `~/klipper/klippy/extras/led_effect.py` → `/home/user/klipper-led_effect/src/led_effect.py`
- `~/klipper/klippy/extras/shaketune` → `/home/user/klippain_shaketune/shaketune` (added 2026-04-10)
- `~/klipper/klippy/extras/autotune_tmc.py` → `/home/user/klipper_tmc_autotune/autotune_tmc.py` (added 2026-04-10)
- `~/klipper/klippy/extras/motor_constants.py` → same repo (added 2026-04-10)
- `~/klipper/klippy/extras/motor_database.cfg` → same repo (added 2026-04-10)
- `~/moonraker/moonraker/components/timelapse.py` → (same pattern for moonraker-timelapse)

This is how plugins install themselves without modifying the upstream Klipper tree. No action needed.

### numpy downgrade on the printer

Shake&Tune's `requirements.txt` downgraded numpy from **2.4.3 → 2.2.2** in `klippy-env` during install. Klipper works fine with either — both are NumPy 2.x — but if anything else in that venv ever needs the newer version, the downgrade is the first thing to check.

### Orca / Klipper acceleration mismatch

OrcaSlicer's printer profile (`orca_slicer/backups/2026-04-10/user/default/machine/ZeroG Mercury One.json:20-27`) declares:

```
"machine_max_acceleration_x": ["40000", "20000"],
"machine_max_acceleration_y": ["40000", "20000"],
```

Klipper is set to `max_accel: 10000` in `printer.cfg:236`. **Klipper wins** — the 40000 in Orca is effectively aspirational and is silently capped. If you want to actually reach the higher ceiling you have to raise the Klipper value (and retest input shaping at the new speeds).

### The "MyKlipper 0.2 nozzle - Copy" stub

`orca_slicer/backups/2026-04-10/user/default/machine/MyKlipper 0.2 nozzle - Copy.json` is 346 bytes — a near-empty template with no real settings. Abandoned accidental copy, safe to delete next time you're in Orca.

### Two filament profiles with different ASA temps

- `user/default/filament/eSun ASA Plus.json` — 250 °C, max vol 10, flow 0.965
- `user/default/filament/eSun ASA Plus - Clean.json` — 250 °C, max vol 18, flow 0.96
- `user/default/filament/base/eSUN ASA Plus @MyKlipper 0.4 nozzle.json` — **260 °C**, max vol 18, flow 0.965 (note `eSUN` vs `eSun` capitalisation)

The base profile (260 °C) and the active profile (250 °C) disagree on nozzle temperature. The active one was tuned down, but the base profile is still what gets imported from the system library. If you re-derive a profile from the base, it'll start at 260 °C again.

### Scarf seam everywhere

All four process profiles have scarf seam enabled with 20 steps, `seam_slope_type: all`, `seam_slope_inner_walls: 1`, `seam_slope_conditional: 1`. This dates back to the 10 May 2024 "perfect z seam scarf" OrcaSlicer backup — the tuning has been carried forward through every version upgrade.

### Beacon mesh bounds are asymmetric

`mesh_min 5,36` → `mesh_max 255,240`. The **Y minimum of 36** is because the Beacon sits 36 mm behind the nozzle on the toolhead (Tentacool / Goliath Short mount), so the probe physically can't reach Y<36. This is also why the home position is `safe_z_home: 135, 125` — centred in the probeable area, not the bed centre.

### Pressure advance drift observed in the logs

The active klippy.log reports `extruder: pressure_advance: 0.045000` at the time of capture, but the config file default is 0.025 and the filament profiles are 0.035 (PLA) and 0.04 (ASA). The 0.045 runtime value was probably set by a tuning-tower session or a manual `SET_PRESSURE_ADVANCE ADVANCE=0.045` that wasn't reset. Worth checking next time you start a print.

### Current print stall count

From klippy.log stats: `print_stall: 17` over ~406 k seconds of session uptime (~4.7 days). Extremely low — healthy. A print stall is when the toolhead move queue drains because the host couldn't keep the MCU fed in time.

### Pi 5 runs cool

Current Pi temperature at snapshot time: 44.6 °C idle. Octopus MCU: 36.8 °C. EBB36 MCU: 37.3 °C. Chamber cold: 26.9 °C. Everything comfortably within limits. The 45 °C threshold in the enclosure fan loop is essentially at the Pi's normal idle temperature, so the fans cycle on briefly when things warm up. Bump the trigger to 55 °C if fan cycling is noisy.

---

*End of references. Add new entries in alphabetical order within the relevant section. If a whole new category opens up, add it to the contents and slot it in where it fits.*
