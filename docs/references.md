# Best practices reference library

> Curated index of external documentation, guides, tools, and communities for running and maintaining the ZeroG Mercury One.1 described in `../HARDWARE.md`.
> This file points at authoritative sources — it does not duplicate them. When something is copied inline, it's because we own it (our own configs, our own logs) or it's a stable quote under 25 words.
> Snapshot as of 2026-04-10.

---

## Contents

1. [Start-here: the canonical tuning guides](#start-here-the-canonical-tuning-guides)
2. [Core Klipper ecosystem](#core-klipper-ecosystem)
3. [Slicer — OrcaSlicer](#slicer--orcaslicer)
4. [Probe and bed sensing — Beacon](#probe-and-bed-sensing--beacon)
5. [Hardware-specific docs](#hardware-specific-docs)
6. [CAN bus / Katapult / EBB36](#can-bus--katapult--ebb36)
7. [Klipper plugins we use (or should)](#klipper-plugins-we-use-or-should)
8. [Safety and chamber management](#safety-and-chamber-management)
9. [Communities and Discord servers](#communities-and-discord-servers)
10. [MCP servers for 3D printing](#mcp-servers-for-3d-printing)
11. [Local notes and gotchas](#local-notes-and-gotchas)

---

## Start-here: the canonical tuning guides

These are the references to consult first for any print-quality or tuning question. If a quality issue doesn't have a clear answer in one of these three, *then* go wider.

| Reference | Maintained by | Why it's first-line |
|---|---|---|
| **[Ellis' Print Tuning Guide](https://ellis3dp.com/Print-Tuning-Guide/)** ([GitHub](https://github.com/AndrewEllis93/Print-Tuning-Guide)) | Andrew Ellis | The reference. Covers extruder calibration, PA, first layer, retraction, flow, VFAs, troubleshooting. Explicitly Klipper-oriented. Our `TEST_SPEED` macro is his. |
| **[Voron Print Tuning Guide](https://docs.vorondesign.com/community/howto/ellis/print-tuning-guide.html)** | Voron Design community | Same Ellis guide mirrored inside the Voron docs, which also host their own tuning section at [docs.vorondesign.com/tuning](https://docs.vorondesign.com/tuning/). Use the Voron side for CoreXY-specific pointers like de-racking. |
| **[OrcaSlicer Calibration Wiki](https://github.com/OrcaSlicer/OrcaSlicer/wiki/Calibration)** | OrcaSlicer project | Slicer-side calibration wizard walkthrough. Recommended order is **temperature → flow → pressure advance → max volumetric speed → retraction**. Each has a built-in print in Orca. |

### Recommended calibration order (synthesis)

1. **Mechanical first** (before touching any numbers) — belts tensioned and equal, rails clean and lubricated, frame de-racked, Z steppers isolated from frame rack by `z_tilt`, toolhead screws tight.
2. **Extruder rotation_distance** — `M83 / G1 E100 F100`, measure. Target 100 ± 0.5 mm.
3. **Hotend PID** — `PID_CALIBRATE HEATER=extruder TARGET=<print temp>`, then `SAVE_CONFIG`.
4. **Bed PID** — same, `HEATER=heater_bed TARGET=<print temp>`.
5. **Chamber PID** — `PID_CALIBRATE HEATER=chamber TARGET=50` (our HotBox is PID-controlled).
6. **Resonance testing** — `SHAPER_CALIBRATE`, copy results into `save_config`. For CoreXY use **Shake&Tune** (see below) rather than raw `TEST_RESONANCES` — it's dramatically better at diagnosing belt/bearing asymmetries.
7. **Per-filament flow rate** — Orca's flow test (coarse then fine).
8. **Per-filament pressure advance** — Orca's PA tower, after flow.
9. **Per-filament max volumetric speed** — Orca test, find the rate where surface quality breaks down.
10. **Per-filament temperature tower** — only if bridging/retraction issues persist.
11. **Retraction** — usually last, once PA and flow are stable.

---

## Core Klipper ecosystem

| Reference | URL | When to consult |
|---|---|---|
| Klipper documentation | [klipper3d.org](https://www.klipper3d.org/) | Any config question, pin reference, G-code command reference |
| Klipper config reference | [klipper3d.org/Config_Reference.html](https://www.klipper3d.org/Config_Reference.html) | Every field, every module |
| Klipper G-codes reference | [klipper3d.org/G-Codes.html](https://www.klipper3d.org/G-Codes.html) | Macro authoring, runtime commands |
| Klipper Discourse forum | [klipper.discourse.group](https://klipper.discourse.group/) | Troubleshooting, community help — more active than GitHub issues |
| Klipper GitHub | [github.com/Klipper3d/klipper](https://github.com/Klipper3d/klipper) | Firmware source, issue tracker |
| Moonraker documentation | [moonraker.readthedocs.io](https://moonraker.readthedocs.io/) | API reference, component config, update manager |
| Moonraker API endpoints | [moonraker.readthedocs.io/en/latest/web_api/](https://moonraker.readthedocs.io/en/latest/web_api/) | Building integrations, automating the printer |
| Mainsail docs | [docs.mainsail.xyz](https://docs.mainsail.xyz/) | Web UI customisation, macros panel, dashboards |
| mainsail-config repo | [github.com/mainsail-crew/mainsail-config](https://github.com/mainsail-crew/mainsail-config) | The `mainsail.cfg` include — standard macro/UI helpers |
| KlipperScreen docs | [klipperscreen.readthedocs.io](https://klipperscreen.readthedocs.io/) | Touchscreen UI, themes, panel customisation |
| Kiauh (installer) | [github.com/dw-0/kiauh](https://github.com/dw-0/kiauh) | Managing Klipper/Moonraker/Mainsail installs, service restarts |
| Klipper save_config format | [klipper3d.org/Config_Checks.html](https://www.klipper3d.org/Config_Checks.html) | Understanding the `#*#` auto-generated block |

---

## Slicer — OrcaSlicer

| Reference | URL | Notes |
|---|---|---|
| **OrcaSlicer Wiki home** | [github.com/OrcaSlicer/OrcaSlicer/wiki](https://github.com/OrcaSlicer/OrcaSlicer/wiki) | Full wiki — the authoritative source, living with the codebase |
| Calibration landing page | [wiki/Calibration](https://github.com/OrcaSlicer/OrcaSlicer/wiki/Calibration) | Index to every calibration test |
| Adaptive PA calibration | [wiki/adaptive-pressure-advance-calib](https://github.com/OrcaSlicer/OrcaSlicer/wiki/adaptive-pressure-advance-calib) | The cooler Klipper-specific approach — PA varies by accel. We don't use this yet |
| Max volumetric speed | [wiki/volumetric-speed-calib](https://github.com/OrcaSlicer/OrcaSlicer/wiki/volumetric-speed-calib) | Relevant because our ASA profile caps at 10 mm³/s while PLA is at 24 — one of these is probably stale |
| Obico OrcaSlicer guides | [obico.io/blog](https://www.obico.io/blog/orcaslicer-comprehensive-calibration-guide/) | Third-party blog that explains the same material more narratively — useful as a cross-reference |
| OrcaSlicer GitHub | [github.com/OrcaSlicer/OrcaSlicer](https://github.com/OrcaSlicer/OrcaSlicer) | Issue tracker, release notes |
| OrcaSlicer Discord | Linked from repo | Community help, profile sharing |

---

## Probe and bed sensing — Beacon

Our printer uses a Beacon RevH as both Z probe and input-shaper accelerometer. All relevant references:

| Reference | URL | Notes |
|---|---|---|
| Beacon documentation portal | [docs.beacon3d.com](https://docs.beacon3d.com/) | Official vendor docs |
| Beacon quick start | [docs.beacon3d.com/quickstart/](https://docs.beacon3d.com/quickstart/) | Installation, first-calibration flow |
| Beacon commands reference | [docs.beacon3d.com/commands/](https://docs.beacon3d.com/commands/) | `BEACON_CALIBRATE`, `BEACON_AUTO_CALIBRATE`, model management |
| Beacon config reference | [docs.beacon3d.com/config/](https://docs.beacon3d.com/config/) | All parameters — `x_offset`, `y_offset`, `mesh_main_direction`, `mesh_runs` |
| Beacon Klipper plugin | [github.com/beacon3d/beacon_klipper](https://github.com/beacon3d/beacon_klipper) | The Klipper extras `beacon.py` we symlink into `~/klipper/klippy/extras/beacon.py` |
| Beacon-KAMP (adaptive mesh) | [github.com/YanceyA/Beacon-KAMP](https://github.com/YanceyA/Beacon-KAMP) | Klipper Adaptive Mesh and Purging integration — mesh only the area your print touches. Considered community best practice for fast prints |
| RevH installation (deep dive) | [deepwiki.com — Plus4-Wiki 2.1](https://deepwiki.com/qidi-community/Plus4-Wiki/2.1-beacon3d-revh-installation-guide) | Third-party RevH install guide — useful for wiring and mounting geometry |
| Klipper Discourse: Beacon setup | [klipper.discourse.group — Calibration & Installation of Beacon Probe](https://klipper.discourse.group/t/calibration-installation-of-beacon-probe/21720) | Community troubleshooting thread |

**Our current Beacon state**: firmware 2.1.0, RevH hardware, `y_offset: 36` (Tentacool / Goliath Short duct), saved calibration model in `backups/2026-04-10/config/printer.cfg:443-457` (model_temp 25.31 °C, model_offset −0.09 mm).

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
| Vz-Printhead in Vz330 build | [docs.vzbot.org/vz330_mellow/gantry/printhead](https://docs.vzbot.org/vz330_mellow/gantry/printhead) | Section 4.3 — integrated build context |
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

### Extruder — LDO Orbiter 2.5

| Reference | URL | Notes |
|---|---|---|
| Orbiter Projects site | [orbiterprojects.com](https://www.orbiterprojects.com/) | Official Orbiter project |
| LDO Motors Orbiter 2 + 2.5 product | [ldomotors.com](https://www.ldomotors.com/) | LDO is the canonical manufacturer |
| Orbiter 2 Klipper config examples | [github.com/orbiter3dio/Orbiter-2](https://github.com/orbiter3dio/Orbiter-2) | Rotation distances, default currents |

**Our config**: `rotation_distance: 4.637`, `full_steps_per_rotation: 200`, run_current 0.85 A, pressure_advance 0.025 default (overridden per-filament).

### Bed heater — Fabreeko Honey Badger

| Reference | URL |
|---|---|
| Fabreeko — Honey Badger product page | [fabreeko.com](https://www.fabreeko.com/) (search "Honey Badger 600W") |
| 110/220 V silicone AC heater, 600 W | Our spec |

### Mainboard — BTT Octopus Pro V1.0

| Reference | URL |
|---|---|
| **BTT Octopus Pro V1.0 repo (pinout, schematic, manual)** | [github.com/bigtreetech/BIGTREETECH-OCTOPUS-Pro](https://github.com/bigtreetech/BIGTREETECH-OCTOPUS-Pro) |
| Klipper Octopus Pro sample config | [bigtreetech/BIGTREETECH-OCTOPUS-V1.0/blob/master/Firmware/Klipper/](https://github.com/bigtreetech/BIGTREETECH-OCTOPUS-V1.0/tree/master/Firmware/Klipper) |
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
| BTT TMC5160T Pro product | [github.com/bigtreetech/BIGTREETECH-TMC5160T-PRO-V1.0](https://github.com/bigtreetech/BIGTREETECH-TMC5160T-PRO-V1.0) | Schematic, dimensions, jumper settings |
| TMC2209 datasheet | [analog.com — TMC2209](https://www.analog.com/en/products/tmc2209.html) | Same for the Z and extruder drivers |
| Klipper TMC drivers doc | [klipper3d.org/TMC_Drivers.html](https://www.klipper3d.org/TMC_Drivers.html) | Sensorless homing, stealthChop, coolStep, driver parameter explanations |

### Steppers

| Reference | URL | Notes |
|---|---|---|
| LDO 42STH48-2804AC datasheet | [ldomotors.com](https://www.ldomotors.com/products/ldo-42sth48-2804ac-super-power-stepper-motor-high-torque) | 2.8 A peak, 0.55 Nm — our X/Y motors |
| Wotrees 17HS4401 | Search Amazon / AliExpress | 1.5 A, 0.42 Nm — our Z motors |

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

## Klipper plugins we use (or should)

### Installed

| Plugin | URL | Role |
|---|---|---|
| `klipper-led_effect` (julianschill) | [github.com/julianschill/klipper-led_effect](https://github.com/julianschill/klipper-led_effect) | LED effects engine — drives our 29-element neopixel chain via `led_marcos.cfg` [sic] |
| `beacon_klipper` (beacon3d) | [github.com/beacon3d/beacon_klipper](https://github.com/beacon3d/beacon_klipper) | Beacon probe driver — dev channel |
| `moonraker-obico` | [github.com/TheSpaghettiDetective/moonraker-obico](https://github.com/TheSpaghettiDetective/moonraker-obico) | Remote monitoring, AI print-failure detection |
| `moonraker-timelapse` | [github.com/mainsail-crew/moonraker-timelapse](https://github.com/mainsail-crew/moonraker-timelapse) | Timelapse generation — half-wired (symlinked config, `[update_manager]` entry commented out) |
| `crowsnest` | [github.com/mainsail-crew/crowsnest](https://github.com/mainsail-crew/crowsnest) | Webcam stack — ustreamer + camera-streamer orchestration |
| `sonar` | [github.com/mainsail-crew/sonar](https://github.com/mainsail-crew/sonar) | Wi-Fi keep-alive — enabled but currently not running |

### Recommended — we should install these

| Plugin | URL | Why |
|---|---|---|
| **Klipper TMC Autotune** (andrewmcgr) | [github.com/andrewmcgr/klipper_tmc_autotune](https://github.com/andrewmcgr/klipper_tmc_autotune) | Auto-tunes TMC driver internal parameters (TBL, HEND, HSTRT, TOFF) per-motor. More accurate than the hand-picked values we currently have on Z. |
| **Shake&Tune (klippain-shaketune)** (Frix-x) | [github.com/Frix-x/klippain-shaketune](https://github.com/Frix-x/klippain-shaketune) | **Strongly recommended for CoreXY**. Adds `COMPARE_BELTS_RESPONSES` to diagnose belt tension asymmetry and `CREATE_VIBRATIONS_PROFILE` to measure vibrations vs speed/direction. Uses the Beacon as accelerometer. Massive upgrade over raw `TEST_RESONANCES`. |
| **KAMP (Klipper Adaptive Meshing & Purging)** | [github.com/kyleisah/Klipper-Adaptive-Meshing-Purging](https://github.com/kyleisah/Klipper-Adaptive-Meshing-Purging) | Meshes only the bed area your print touches — saves time on small prints. Combine with [Beacon-KAMP](https://github.com/YanceyA/Beacon-KAMP) for Beacon-specific optimisations. |
| **dockable_probe / automatic_fan_control / etc.** | See Klipper extras | Not needed for us currently but worth knowing about |

---

## Safety and chamber management

Because this printer has a heated enclosure (HotBox up to 70 °C), a 600 W AC bed heater on an SSR, no door interlock, and no exhaust, safety discipline matters.

| Topic | Reference |
|---|---|
| Thermal runaway in Klipper | [klipper3d.org/Config_Reference.html#verify_heater](https://www.klipper3d.org/Config_Reference.html#verify_heater) — we use `verify_heater chamber` with `max_error 600, check_gain_time 300, hysteresis 5, heating_gain 0.5` |
| Snapmaker — 3D printer fire safety | [snapmaker.com/blog — fire-safety-causes-prevention](https://www.snapmaker.com/blog/3d-printer-fire-safety-causes-prevention-best-practices/) |
| Alveo3D — 3D printer fires | [alveo3d.com/en/3d-printer-fire-safety](https://www.alveo3d.com/en/3d-printer-fire-safety/) |
| JLC3DP — fire safety risks and protection | [jlc3dp.com/blog/3d-printer-fire-safety](https://jlc3dp.com/blog/3d-printer-fire-safety) |
| Voron ABS/ASA enclosure printing tips | [docs.vorondesign.com](https://docs.vorondesign.com/) (see "Tips" section) |
| Qidi Tech — thermal runaway prevention | [qidi3d.com/blogs/news/prevent-thermal-runaway-3d-printer](https://qidi3d.com/blogs/news/prevent-thermal-runaway-3d-printer) |
| 3DPUT — heated enclosures | [3dput.com/heated-3d-printer-enclosures](https://3dput.com/heated-3d-printer-enclosures/) |

### Safety checklist (synthesis — non-exhaustive)

- [ ] Klipper `[verify_heater]` enabled on every heater (currently: hotend via default, bed via default, chamber explicit)
- [ ] Physical thermal fuse on the HotBox element (**yes** — 125 °C ceramic, `github.com/nark3d/HotBox` BOM)
- [ ] Smoke/fire alarm in the room (**operational — confirm**)
- [ ] Non-flammable surface under printer (**operational — confirm**)
- [ ] VOC ventilation for ASA/ABS (**no — TBD**; currently no exhaust fan and no door interlock — long ASA prints should only run with supervision or a windowed room)
- [ ] SSR heat dissipation — Heschen SSR-40DA driving 600 W @ 240 V ≈ 2.5 A is comfortably inside spec but benefits from a finned heatsink (**TBD — confirm installed**)
- [ ] Mains-side fused protection upstream of the SSR (**TBD — confirm**)
- [ ] Physical kill switch (**yes** — BTT Relay V1.2 with 10 A rocker)

---

## Communities and Discord servers

| Community | Link | Focus |
|---|---|---|
| **ZeroG Discord** | Invite from [docs.zerog.one](https://docs.zerog.one/) | Mercury One.1, Nebula, Hydra, Electronics Enclosure — the primary build support community |
| **VzBot Discord** | Invite from [docs.vzbot.org](https://docs.vzbot.org/) | VzBot printers and VzBot printheads (our toolhead) |
| **Voron Discord** | Invite from [vorondesign.com](https://vorondesign.com/) | General CoreXY / Klipper expertise — even for non-Voron printers |
| **Beacon Discord** | Linked from [docs.beacon3d.com](https://docs.beacon3d.com/) | Beacon-specific support |
| Klipper Discourse | [klipper.discourse.group](https://klipper.discourse.group/) | Official Klipper forum — best for deep troubleshooting |
| r/klippers | [reddit.com/r/klippers](https://www.reddit.com/r/klippers/) | General Klipper community |
| r/voroncorexy | [reddit.com/r/voroncorexy](https://www.reddit.com/r/voroncorexy/) | CoreXY community |
| r/3Dprinting | [reddit.com/r/3Dprinting](https://www.reddit.com/r/3Dprinting/) | Everything 3D printing |

---

## MCP servers for 3D printing

These are Model Context Protocol servers that let AI assistants (Claude, etc.) talk directly to printers or adjacent tooling. Relevant because Claude Code can mount them to query/control the Mercury One.1 live.

| Server | Scope | Maturity | Notes |
|---|---|---|---|
| **DMontgomery40/mcp-3D-printer-server** | Multi-protocol: Moonraker, OctoPrint, Duet, Bambu, Prusa Connect, Creality Cloud, Repetier | **173 stars**, GPL v2, npm-installable (`npm install -g mcp-3d-printer-server`), Docker support, TypeScript 4.9+, Node 18+ | [GitHub](https://github.com/DMontgomery40/mcp-3D-printer-server) · [npm](https://www.npmjs.com/package/mcp-3d-printer-server) · [Awesome MCP listing](https://mcpservers.org/servers/DMontgomery40/mcp-3D-printer-server). Supports status, gcode send, file upload/list, temperature control, STL manipulation, slicing. Security: dangerous ops require `ARMED=True`, destructive ops need admin PIN, audit log. **This is the mature option.** |
| Charleslotto/klipper-mcp | Klipper via Moonraker | Early / VS Code-oriented | [Glama listing](https://glama.ai/mcp/servers/@Charleslotto/klipper-mcp) · [LobeHub](https://lobehub.com/mcp/charleslotto-klipper-mcp) |
| grego33/klipper-config-mcp | Klipper config editing | Early | [GitHub](https://github.com/grego33/klipper-config-mcp) |
| OctoEverywhere MCP | Status/control via OctoEverywhere cloud | Vendor | [Blog post](https://blog.octoeverywhere.com/mcp-server-for-3d-printing/) — requires OctoEverywhere account |

### If we install DMontgomery40/mcp-3D-printer-server

The printer side just needs Moonraker accessible at `http://192.168.0.37:7125`. Environment config for Claude Code:

```
PRINTER_TYPE=klipper
PRINTER_HOST=192.168.0.37
PRINTER_PORT=7125
API_KEY=<optional if Moonraker authorization allows trusted_clients>
```

Since Moonraker's `[authorization]` block currently trusts `192.168.0.0/16` (see `backups/2026-04-10/config/moonraker.conf:19-27`), Claude Code running on your Mac (192.168.0.43) is already on the trusted list — **no API key needed**.

Capabilities this would unlock inside Claude Code:
- Read live printer state (temps, position, print progress, filename, etc.)
- Read `klippy.log` live without re-rsyncing
- Send gcode commands (gated behind `ARMED=True` for safety)
- Upload sliced gcode directly from Claude → Moonraker
- Query bed mesh data
- Emergency stop
- STL manipulation (scaling, rotation, sectioning) via the server's built-in helpers
- Slicing via a bundled slicer path

**Proposing**: install this as part of the groundwork. Leaving in `ARMED=False` mode means Claude can *read* everything but can't *change* anything — perfect for the observational phase of any project.

---

## Local notes and gotchas

Things specific to this machine that you'd otherwise forget:

### Klipper repo is "dirty"

Klippy.log shows the Klipper version as `v0.13.0-540-g57c2e0c96-dirty`. This is **not a problem** — the "dirty" flag is caused by two **untracked symlinks** in the working tree:

- `~/klipper/klippy/extras/led_effect.py` → `/home/adam/klipper-led_effect/src/led_effect.py`
- `~/moonraker/moonraker/components/timelapse.py` → (same pattern for moonraker-timelapse)

This is how both plugins install themselves without modifying the upstream Klipper tree. No action needed.

### Orca slicer / Klipper acceleration mismatch

OrcaSlicer's printer profile (`orca_slicer/backups/2026-04-10/user/default/machine/ZeroG Mercury One.json:20-27`) declares:

```
"machine_max_acceleration_x": ["40000", "20000"],
"machine_max_acceleration_y": ["40000", "20000"],
```

Klipper is set to `max_accel: 10000` in `printer.cfg:236`. **Klipper wins** — the 40000 in Orca is effectively aspirational and is silently capped. If you want to actually reach the higher ceiling you have to raise the Klipper value (and retest input shaping at the new speeds).

### The "MyKlipper 0.2 nozzle - Copy" stub

`orca_slicer/backups/2026-04-10/user/default/machine/MyKlipper 0.2 nozzle - Copy.json` is 346 bytes — a near-empty template with no real settings. It's an abandoned accidental copy, not an active profile. Safe to delete next time you're in Orca.

### Two filament profiles with different ASA temps

- `user/default/filament/eSun ASA Plus.json` — 250 °C, max vol 10, flow 0.965
- `user/default/filament/eSun ASA Plus - Clean.json` — 250 °C, max vol 18, flow 0.96
- `user/default/filament/base/eSUN ASA Plus @MyKlipper 0.4 nozzle.json` — **260 °C**, max vol 18, flow 0.965 (note `eSUN` vs `eSun` capitalisation)

The base profile (260 °C) and the active profile (250 °C) disagree on nozzle temperature. You tuned the active one down, but the base profile is still what gets imported from the system library. If you re-derive a profile from the base, it'll start at 260 °C again.

### Scarf seam everywhere

All four process profiles have scarf seam enabled with 20 steps, `seam_slope_type: all`, `seam_slope_inner_walls: 1`, `seam_slope_conditional: 1`. This dates back to the 10 May 2024 "perfect z seam scarf" OrcaSlicer backup — the tuning has been carried forward through every version upgrade.

### Beacon mesh bounds are asymmetric

`mesh_min 5,36` → `mesh_max 255,240`. The **Y minimum of 36** is because the Beacon sits 36 mm behind the nozzle on the toolhead (Tentacool / Goliath Short mount), so the probe physically can't reach Y<36. This is also why the home position is `safe_z_home: 135, 125` — centred in the probeable area, not the bed centre.

### Pressure advance drift observed in the logs

The active klippy.log reports `extruder: pressure_advance: 0.045000` at the time of capture, but the config file default is 0.025 and the filament profiles are 0.035 (PLA) and 0.04 (ASA). The 0.045 runtime value was probably set by a tuning-tower session or a manual `SET_PRESSURE_ADVANCE ADVANCE=0.045` that wasn't reset. Worth checking next time you start a print — if you see PA values drift outside the filament profile's range, something in the macro chain isn't restoring state cleanly.

### Current active print stall count

From the klippy.log stats tail: `print_stall: 17` over ~406k seconds of session uptime (~4.7 days). That's a very low stall rate — healthy. A print stall is when the toolhead move queue drains because the host couldn't keep the MCU fed in time. Over 4.7 days of operation, 17 is background noise.

### Pi 5 runs cool

Current Pi temperature: 44.6 °C idle. Octopus MCU: 36.8 °C. EBB36 MCU: 37.3 °C. Chamber cold: 26.9 °C. Everything is comfortably within limits — the 45 °C threshold in the enclosure fan loop is essentially at the Pi's normal idle temperature, so the fans will cycle on briefly when things warm up. Consider bumping that trigger up to 55 °C if fan cycling is noisy.

---

*End of references. To add more: drop in alphabetical order within the relevant section. If a whole new category opens up (e.g., "Print farm integration"), add it after Safety and before Communities.*
