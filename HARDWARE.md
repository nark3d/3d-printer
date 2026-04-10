# Hardware reference — ZeroG Mercury One.1 "Beast"

> Snapshot as of 2026-04-10. Describes the physical machine: components, wiring, and which config file governs each subsystem.
> For the matching software state see `backups/2026-04-10/` (printer / Klipper / Moonraker) and `orca_slicer/backups/2026-04-10/` (slicer profiles).

---

## Contents

1. [At a glance](#at-a-glance)
2. [Lineage and identity](#lineage-and-identity)
3. [Frame and enclosure](#frame-and-enclosure)
4. [Motion system](#motion-system)
5. [Hotbed](#hotbed)
6. [Toolhead](#toolhead)
7. [Control electronics](#control-electronics)
8. [Chamber conditioning](#chamber-conditioning)
9. [Lighting](#lighting)
10. [UI and remote access](#ui-and-remote-access)
11. [Filament path](#filament-path)
12. [Software stack](#software-stack)
13. [Pin assignment reference](#pin-assignment-reference)
14. [Bill of materials](#bill-of-materials)
15. [Known gaps and future work](#known-gaps-and-future-work)

---

## At a glance

| | |
|---|---|
| Display name | "Zero G Beast" (set in Mainsail `backup-mainsail.json:1`) |
| Type | Enclosed CoreXY, direct-drive, 3-point Z with auto-level |
| Base design | ZeroG **Mercury One.1** (CoreXY conversion of Ender 5 Pro) |
| Frame kit | ZeroG **Nebula Pro** (LDO Motors, pre-cut anodised extrusions) |
| Bed module | ZeroG **Hydra** — 275×275 variant |
| Electronics housing | ZeroG **Electronics Enclosure** (bottom compartment) |
| Usable print volume | ~260 × 245 × 258 mm (X / Y / Z) |
| Kinematics | CoreXY, 3 × independent Z motors with `z_tilt` auto-levelling |
| Host | Raspberry Pi 5 (8 GB) running Debian Bookworm |
| Primary MCU | BigTreeTech **Octopus Pro V1.0** (STM32F429ZGT6) |
| Toolhead MCU | BigTreeTech **EBB36 Gen2** (STM32G0B1) — currently over USB (CAN provisioned but not live) |
| Probe | **Beacon RevH** eddy-current contactless — also the input-shaper accelerometer |
| Tuned ceiling | `max_velocity` 600 mm/s, `max_accel` 10000 mm/s², MZV input shaper X=63.8 Hz, Y=43.4 Hz |
| Chamber heat | Custom 200 W PTC **HotBox** (designed by the owner, open source) |
| Air filtration | **BentoBox** carbon filter with 4028 fans |
| Screen | Waveshare 5″ capacitive DSI touchscreen |
| Remote | Mainsail + Moonraker-Obico |
| Network | Wi-Fi only, `3d-printer` on 192.168.0.37 |

---

## Lineage and identity

This is not a single off-the-shelf printer. It is a **ZeroG Mercury One.1** (a Voron-adjacent CoreXY conversion kit for the Ender 5 Pro) stacked with **three additional ZeroG sub-projects** and a pile of upgrades accumulated across three rebuilds.

ZeroG sub-projects in play ([docs.zerog.one](https://docs.zerog.one/)):

| Project | What it does | Relevant link |
|---|---|---|
| **Mercury One.1** | The CoreXY conversion — replaces the Ender 5 Pro gantry, retains the base PSU and some extrusions | [docs.zerog.one/manual/build/mercury_eva](http://docs.zerog.one/manual/build/mercury_eva) |
| **Nebula Frame Kit (Pro)** by LDO | Complete drop-in replacement extrusion kit — pre-cut, pre-tapped, anodised. Pro size suits the 275 mm bed | [kb-3d Nebula Pro](https://kb-3d.com/store/frame-enclosure/3971-zero-g-mercury-one1-nebula-frame-kit-multiple-sizes-colors.html), [Fabreeko](https://www.fabreeko.com/products/zero-g-mercury-nebula-frames-by-ldo) |
| **Hydra heated bed** | Separate bed-carrier project — 4 variants, one is the 275×275 Ender-5-Pro option this printer uses | [docs.zerog.one/manual/build/hydra/heated_bed_drawing](http://docs.zerog.one/manual/build/hydra/heated_bed_drawing) |
| **Electronics Enclosure** | Bottom housing for Octopus / Pi / PSU / AC switching | [docs.zerog.one/manual/build/electronics_enclosure](http://docs.zerog.one/manual/build/electronics_enclosure) |

**Three-phase history** (from this repository's git log + dated config snapshots):

| Phase | When | Characterised by |
|---|---|---|
| Phase 1 | Mar 2024 (`printercfgs/`) | Still Cartesian, single Z, 200×200×200, max_velocity 300, max_accel 2000 — basically stock-ish Ender 5 Pro |
| Phase 2 | Apr 2024 (`backups/2024-04-*/`) | First CoreXY conversion — kinematics changed, triple Z with `z_tilt`, max_velocity 600, max_accel 15000, BLTouch probe, PT1000 hotend, input shaper tuned |
| Phase 3 | Feb–Apr 2026 (`backups/2026-04-10/`) | Full rebuild: Nebula frame, Hydra bed, VzBot CNC toolhead, Beacon probe replacing BLTouch, EBB36 CAN toolhead board (currently over USB), HotBox chamber heater, BentoBox filter, aluminium upgrade parts throughout |

The pre-Phase-3 config is preserved on the printer at `~/printer_data/config-backup-pre-canbus.tar` (Mar 22 2026), also copied to `backups/2026-04-10/misc/`.

---

## Frame and enclosure

| Component | Spec |
|---|---|
| Frame kit | ZeroG **Nebula Pro** by LDO Motors |
| Extrusions | 2020 and 2040 mix (per Nebula Pro kit — see ZeroG docs for full list) |
| Panels | 3 mm transparent acrylic |
| Door | No interlock, no safety switch |
| Top exhaust | None |
| Mounting accessories | "All-aluminium" hotend mount, bed arms, stepper motor mounts (upgraded from printed parts) |

The bottom compartment is the ZeroG Electronics Enclosure which houses the Octopus Pro, Pi 5, Meanwell PSU, SSR, BTT safety relay, and the two 25 A MOSFET modules. Not a dedicated cooling fan bank — cooling is handled by the `enclosure_fans` output (see [Chamber conditioning](#chamber-conditioning)).

> **Overall outer frame dimensions**: not yet measured — see the ZeroG Nebula Pro docs for the kit's stock dimensions. Approximate from photo: ~420 × 420 × 640 mm. **TBD** — update when measured.

---

## Motion system

### Kinematics (`backups/2026-04-10/config/printer.cfg:233-238`)

```
kinematics: corexy
max_velocity: 600
max_accel: 10000
max_z_velocity: 5
max_z_accel: 50
```

CoreXY with dual-belt anchoring at the front (ZeroG-style 4-anchor termination). The four belts are **GT2 6 mm**.

### X / Y axes

| | X | Y |
|---|---|---|
| Driver | BTT **TMC5160T Pro** V1.0 (SPI stepstick w/ heatsink) | BTT **TMC5160T Pro** V1.0 |
| Stepper | **LDO-42STH48-2804AC** Super Power (2.8 A peak, 0.55 Nm) | **LDO-42STH48-2804AC** |
| Run current | 2.24 A | 2.24 A |
| Sense resistor | 0.075 Ω | 0.075 Ω |
| `interpolate` | true | true |
| `driver_SGT` | −64 (sensorless-capable) | — |
| Sensorless home | — | `diag0_pin: !PG9` → Y sensorless |
| X endstop | Physical switch on EBB36 at `^!EBB36:PA15` (toolhead-mounted filament/endstop port) | — |
| `rotation_distance` | 39.421 (calibrated) | 39.581 (calibrated) |
| `homing_speed` | 300 mm/s | 300 mm/s |
| `position_max` | 260 | 245 |
| Linear rails | MGN12H recommended for 275×275 Nebula Pro size (**TBD — confirm model / length**) | MGN12H (**TBD**) |

X and Y share the SPI software pins (`PA7/PA6/PA5`). The calibrated rotation_distance values (not the nominal 40.0) indicate end-to-end belt/pulley correction has been performed.

### Z axis

Three independent Z steppers driven by individual TMC2209 stepsticks, auto-levelled by `[z_tilt]`.

| | Z | Z1 | Z2 |
|---|---|---|---|
| Driver | BTT TMC2209 V1.3 UART stepstick | BTT TMC2209 V1.3 | BTT TMC2209 V1.3 |
| Stepper | Wotrees NEMA 17 **17HS4401** (1.5 A / 0.42 Nm) | same | same |
| Run current | 1.15 A | 1.15 A | 1.15 A |
| `driver_TBL / HEND / HSTRT / TOFF` | 0 / 6 / 7 / 4 | same | same |
| `rotation_distance` | 2.002 (individually calibrated) | 1.990 | 1.990 |
| Lead screw | Sourcingmap **Tr8×2** (2 mm pitch, 8 mm lead, 350 mm stainless) | same | same |
| Nut | Brass T8 anti-backlash | same | same |
| Coupler | **Oldham coupler** (compensates for shaft misalignment) | same | same |

The three-decimal-place rotation_distance drift between motors is the direct evidence of Oldham compensation — the slight mechanical difference between axes is absorbed in software after calibration.

**Z tilt points** (`printer.cfg:365-378`):

```
z_positions:
    130, 245    # rear middle, stepper_z
    0, 0        # front left, stepper_z1
    260, 0      # front right, stepper_z2
points:
    130, 209    # rear middle probe point
    10, 0       # front left probe point
    250, 0      # front right probe point
```

`retry_tolerance: 0.01`, up to 5 retries at `horizontal_move_z: 10`, `speed: 600`.

### Endstops and homing

| Axis | Endstop | Notes |
|---|---|---|
| X | `^!EBB36:PA15` (physical switch on toolhead) | Moved to the EBB36 during Phase 3 |
| Y | `!PG9` via `diag0_pin` on TMC5160 (sensorless) | — |
| Z | `probe:z_virtual_endstop` (Beacon) | `safe_z_home` at 135, 125 |

---

## Hotbed

### Plate

| Spec | Value |
|---|---|
| Size | 275 × 275 mm (ZeroG Hydra 275 variant for Ender 5 Pro) |
| Material | Solid aluminium |
| Thickness | **10 mm** (top of ZeroG Hydra's recommended 8–10 mm range) |
| Surface | **PEI** on magnetic stick-on carrier sheet (swappable — multiple sheets owned) |

Reference drawing: [docs.zerog.one/manual/build/hydra/heated_bed_drawing](http://docs.zerog.one/manual/build/hydra/heated_bed_drawing)

### Heater

| Spec | Value |
|---|---|
| Heater | **Fabreeko Honey Badger 600 W** silicone AC heater |
| Voltage | 110 / 220 V AC |
| Switching | SSR (see below) |
| Klipper config | `heater_pin: PA1`, `sensor_type: Generic 3950` on `PF3`, watermark-then-PID (current saved PID: Kp 21.945 / Ki 1.068 / Kd 112.742) |
| Max temp | 130 °C |

### Power switching

| Component | Spec |
|---|---|
| Solid State Relay | **Heschen SSR-40DA**, 4–32 V DC control, 24–480 V AC load, 40 A, 50/60 Hz single-phase |
| Mounting | Electronics enclosure (no dedicated heatsink **TBD — confirm**) |

The SSR control pin is driven from the Octopus `PA1` bed heater output. The Honey Badger's 600 W at 240 V draws ~2.5 A AC — well inside the SSR's 40 A rating.

### Bed mounting (from Hydra docs)

Kinematic-style mount: the 10 mm plate sits on **Kossel balls** resting on three aluminium bed arms, mechanically captured by bolt-downs to prevent movement while allowing thermal expansion. The three-point geometry is what makes `z_tilt` meaningful — the bed is compensated for frame rack, not for a warped plate.

### Thermal profile

- PID controlled (see values above)
- Last bed mesh: 10×10 bicubic over `mesh_min 5,36 → mesh_max 255,240` — the `y_offset 36` corresponds to the Beacon's Y-mounted offset on the toolhead
- Saved mesh in `printer.cfg:407-430` (`[bed_mesh default]`)

---

## Toolhead

### Mount body

| | |
|---|---|
| Mount | **VzBot CNC toolhead** — the "Zero G edition" aluminium variant |
| Purpose | Holds the hotend, extruder, part cooling duct, and Beacon probe |

### Hotend

| Spec | Value |
|---|---|
| Hotend | **Phaetus Rapido 2 HF** (High Flow, 24 V, black) |
| Heater | Integrated 90 W ceramic **planar** heater (not a cartridge) — heats to 200 °C in ~35 s |
| Thermistor | **PT1000** with 2200 Ω pullup on EBB36 (required for PT1000 on EBB boards) |
| Nozzle | Phaetus hardened steel (standard with the Rapido 2) |
| Silicone sock | Standard Phaetus sock shipped with hotend |
| PID tuned | Kp 15.688 / Ki 0.850 / Kd 72.362 (saved in `ebb_gen2.cfg:28-31`) |
| Max temp | 300 °C (config limit) |

References: [Phaetus Rapido 2 HF on 3DJake](https://www.3djake.com/phaetus/rapido-hotend-2-hf)

### Extruder

| Spec | Value |
|---|---|
| Extruder | **Orbiter 2.5** with stock LDO pancake stepper |
| Gear ratio | 7.5 : 1 (dual-gear BMG-style, Orbiter native) |
| `rotation_distance` | 4.637 (close to Orbiter spec, calibrated) |
| Filament | 1.75 mm |
| Driver | TMC2209 onboard EBB36 at 0.85 A run / 0.10 A hold |
| Pressure advance | 0.025 default (per-filament: 0.035 PLA, 0.04 ASA) |

### Probe and mounting duct

| Spec | Value |
|---|---|
| Probe | **Beacon RevH** — contactless eddy-current Z probe |
| Mount | Tentacool / Goliath Short printed ABS/ASA duct |
| Y offset | 36 mm (RevH standard for the Goliath Short mount) |
| X offset | 0 |
| Beacon also serves as | Input shaper accelerometer (`accel_chip: beacon`, probe point 130,130,20) |
| Calibrated model | Saved in `printer.cfg:443-457` — `model_temp 25.31`, `model_offset -0.09` |

Because the Beacon is a contactless sensor, Z homing uses `probe:z_virtual_endstop` across all three Z steppers — no physical Z endstop switch anywhere on the machine.

### Part cooling and heat break

| Fan | Config | Pin | Notes |
|---|---|---|---|
| Part-cooling fan | `[fan]` | `EBB36:PD3` | Standard Orca-controlled fan |
| Heat-break fan | `[heater_fan heatbreak_cooling_fan]` | `EBB36:PA5` | On whenever extruder > 50 °C |

### Accelerometer (onboard, unused)

The EBB36 has an onboard LIS2DW accelerometer that is **not** currently used — Klipper uses the Beacon probe as the accelerometer instead. The LIS2DW lines are present but commented out in `ebb_gen2.cfg:77-86`.

### Estimated toolhead mass

~**550 g** (estimated from components):

- VzBot CNC body: ~220 g
- Rapido 2 HF: ~90 g
- Orbiter 2.5 + LDO pancake stepper: ~135 g
- Beacon RevH: ~15 g
- Duct + 2× toolhead fans: ~55 g
- Wiring umbilical stub: ~30 g

This estimate is why Y=43.4 Hz is a reasonable input shaper frequency: a heavier toolhead + dual-belt 6 mm GT2 geometry biases Y resonance down. X is higher at 63.8 Hz because X drives only the toolhead mass, while Y drives toolhead plus X-gantry.

---

## Control electronics

### Octopus Pro V1.0 (primary MCU)

| Spec | Value |
|---|---|
| Board | BigTreeTech **Octopus Pro V1.0** |
| MCU | STM32F429ZGT6 @ 168 MHz, 1 MB flash |
| Amazon ASIN | B09JC2NR1L — confirmed V1.0 across all storefronts |
| USB serial | `usb-Klipper_stm32f429xx_27001B001250304738313820-if00` |
| Klipper firmware | `v0.13.0-540-g57c2e0c96-dirty` (from `klippy.log`) |
| Compile config | `make menuconfig` → STM32F429, 32 KiB bootloader, 8 MHz crystal |
| Drives | X/Y via TMC5160T Pro stepsticks, 3× Z via TMC2209 stepsticks, bed heater, chamber heater, chamber fan, enclosure fans, BentoBox control, main lights, safety relay, neopixel chain, BLTouch pins (unused) |

> The V1.0 template warning comment at the top of `printer.cfg:1-14` is sample boilerplate from the Klipper repo — it warns *against* using this config on a V1.1 board. On a V1.0 (which this is), it's correct as-is.

### EBB36 Gen2 (toolhead MCU)

| Spec | Value |
|---|---|
| Board | BigTreeTech **EBB36 Gen2 V1.0** |
| MCU | STM32G0B1 @ 64 MHz |
| USB serial | `usb-Klipper_stm32g0b1xx_EBB36_GEN-if00` |
| Connection | **USB over the main umbilical** — CAN capable, Katapult bootloader flashed, but `can0` not configured on the host |
| Drives | Extruder stepper (TMC2209 onboard), hotend heater, part-cooling fan, heat-break fan, hotend PT1000, X endstop switch, driver thermistor, onboard LIS2DW accelerometer (unused) |
| Config | `backups/2026-04-10/config/ebb_gen2.cfg` |

### Beacon RevH (probe MCU)

| Spec | Value |
|---|---|
| Product | **Beacon RevH** contactless eddy-current probe |
| USB serial | `usb-Beacon_Beacon_RevH_080C96DB4E5737374D202020FF100F37-if00` |
| Firmware | `BEACON_REV=H`, accelerometer enabled |
| Role | Z probe + bed mesher + input-shaper accelerometer (one device, three jobs) |

### Raspberry Pi 5 (host)

| Spec | Value |
|---|---|
| Model | Raspberry Pi 5 Model B Rev 1.0 |
| RAM | 8 GB |
| OS | Debian GNU/Linux 12 (bookworm), kernel 6.12.47+rpt-rpi-2712 (2025-09-16) |
| Storage | 117 GB SD card (35 GB used / 77 GB free) |
| Network | Wi-Fi only (eth0 down), `192.168.0.37`, hostname `3d-printer` |
| Temperature reporting | `temperature_sensor raspberry_pi` via `temperature_host` |
| Installed via | **kiauh** v6.1.0 |

### BTT safety relay and power input

| Component | Spec |
|---|---|
| Module | **BigTreeTech Relay V1.2** Power Monitoring Module with included 10 A / 250 V rocker switch |
| Role | AC input monitoring and switchable emergency cut-off |
| Klipper output | `[output_pin relay] pin: PE11`, `value: 1`, `pwm: False` |
| Macros | `TURN_RELAY_ON` / `TURN_RELAY_OFF` in `macros.cfg:38-44` |
| Default state | Powered on at Klipper boot |

This is the primary physical kill switch for the whole printer. There is **no** separate e-stop, no door interlock, and no thermal runaway detection beyond what Klipper provides natively.

### MOSFETs (LED and HotBox)

Two **25 A MOSFET modules** (generic "3D printer heat bed expansion" type) drive the higher-current 24 V accessories:

| Module | Drives | Octopus pin |
|---|---|---|
| MOSFET #1 | HotBox PTC heater (200 W @ 24 V ≈ 8.3 A) | `PA3` via `[heater_generic chamber]` |
| MOSFET #2 | 24 V LED MOSFETs / main lighting — **TBD, confirm exact routing** | PD15 (`output_pin main_lights`) routes via MOSFET? or direct neopixel chain via PB0? |

> **To clarify**: the neopixel chain (PB0) is directly driven from the Octopus 5 V logic line — it does not need a MOSFET. So the two MOSFET modules presumably drive: (1) the HotBox 200 W element (definitely), (2) something else — possibly the chamber lighting if it's a non-addressable 24 V strip, possibly the BentoBox fans. **TBD** — confirm by visual inspection of the electronics enclosure wiring.

### Power supply

| Component | Spec |
|---|---|
| PSU | Stock **Ender 5 Pro** PSU (Meanwell-equivalent, typically 24 V 350 W / LRS-350-24) |
| Voltage rails | 24 V DC (drives everything except the bed heater) |
| Bed heater | Runs on AC mains directly via the Heschen SSR — bypasses the PSU entirely |

### Electronics Enclosure (ZeroG project)

Bottom compartment of the frame. Houses:

- Octopus Pro
- Raspberry Pi 5
- Meanwell PSU
- Heschen SSR
- BTT safety relay
- 2 × 25 A MOSFETs
- AC mains wiring to the Honey Badger bed heater

Wiring from the toolhead (EBB36 USB + Beacon USB) is routed via the drag chain. The bed arm wiring (bed heater AC + thermistor) is separately managed.

---

## Chamber conditioning

### HotBox (owner-designed, open source)

Chamber heater assembly, designed and published by the owner (nark3d on GitHub / Printables).

| Spec | Value |
|---|---|
| Element | 24 V **200 W ceramic PTC**, self-limiting, derate above 87 °C |
| Thermal fuse | **125 °C** ceramic, 250 V / 15 A — soldered inline |
| Fans | **2 × 24 V 4028 axial** — upgraded from 4020 for more airflow. Must run at 100 % whenever element is powered |
| Thermistor | NTC 100 K, thermal-glued (EC360 2 W/mK) into the PTC body |
| Enclosure | Printed housing, physical dimensions 100 mm W × 52 mm D × 177 mm H |
| Mounting | Physically installed on the **right side of the bed** (per photo) |
| Max tested temp | 60 °C chamber — the printer has not been stressed higher |
| Heat-up time (uninsulated enclosure) | ~45 minutes to target |
| Max software limit | 70 °C |
| Published at | [Printables 1636410](https://www.printables.com/model/1636410-hotbox) / [github.com/nark3d/HotBox](https://github.com/nark3d/HotBox) |

**Klipper integration** (`printer.cfg:116-173`):

```
[heater_generic chamber]       # The PTC heater itself
heater_pin: PA3
sensor_pin: PF6                # Chamber top thermistor
control: pid                   # Kp 100, Ki 0.3, Kd 30

[temperature_fan hotbox]       # The twin 4028 fans on the element
pin: PD14
sensor_pin: PF7                # Element body thermistor (for safety loop)
target_temp: 30
```

**Safety loop** — the `HOTBOX_CONTROL` delayed gcode (`printer.cfg:151-173`) runs every 5 s. If the element body temperature ever exceeds 87 °C while the chamber is calling for heat, it cuts the chamber target to 0, starts a 60-second back-off timer, then automatically restarts with the stored target once the element cools back below 70 °C (via `HOTBOX_RESTART` in `hotbox_macros.cfg`). The stored target is persisted in a `gcode_macro HOTBOX_STATE` variable so it survives manual Mainsail adjustments.

**User macros** (`hotbox_macros.cfg`):

- `HOTBOX_ON TARGET=60` — start heating (clamped 30–70 °C)
- `HOTBOX_OFF` — stop heating
- `HOTBOX_STATUS` — report element temp, chamber temp, fan speed, stored target

### BentoBox

| Spec | Value |
|---|---|
| Design | **Voron BentoBox** community print (standard BOM) |
| Fans | 2 × 4028 axial — same as HotBox |
| Filter media | Activated carbon |
| Mounting | **Left side of bed** (per photo) |
| Klipper pin | `[output_pin bentobox] pin: PD13, pwm: True` |
| Control | **Manual via Mainsail** — no automation macro |
| Triggered by | User command (no `gcode_macro` wrapper — toggled by setting the output pin value directly in the UI) |

### Electronics-bay cooling fans

| Spec | Value |
|---|---|
| Klipper pin | `[output_pin enclosure_fans] pin: PD12, pwm: True` |
| Count / model | **TBD — confirm count and model** |
| Location | Inside the ZeroG Electronics Enclosure at the base of the printer |
| Control | Automatic via `ENCLOSURE_FAN_CONTROL` delayed gcode loop |
| Trigger | Turns on at 100 % when Pi temp OR Octopus MCU temp > 45 °C, otherwise off |
| Loop interval | Every 10 seconds |

### Chamber thermistors

| Sensor | Pin | Purpose |
|---|---|---|
| `[heater_generic chamber]` sensor | `PF6` | **Top of chamber** — primary control sensor for the HotBox loop |
| `temperature_sensor chamber_bottom` | `PF5` | **Bottom of chamber** — monitoring only (reported in Mainsail/KlipperScreen titlebar) |
| `temperature_fan hotbox` sensor | `PF7` | PTC element body — safety loop |
| `temperature_sensor Octopus_pro` | — | `temperature_mcu` internal — triggers enclosure fans |
| `temperature_sensor raspberry_pi` | — | `temperature_host` internal — triggers enclosure fans |
| Thermistor type | Generic 3950 for all chamber locations | — |

### Automation loop summary

The printer runs three delayed-gcode control loops simultaneously:

1. **`HOTBOX_CONTROL`** — 5 s tick, element over-temp back-off
2. **`HOTBOX_BACKOFF`** — one-shot, schedules `HOTBOX_RESTART` after 60 s
3. **`ENCLOSURE_FAN_CONTROL`** — 10 s tick, enclosure fan on/off by Pi/MCU temp

---

## Lighting

29-element WS2812-family neopixel chain driven directly from the Octopus Pro (no MOSFET needed — 5 V logic-level).

| | |
|---|---|
| Klipper config | `[neopixel leds] pin: PB0, chain_count: 29, color_order: GRB` |
| Plugin | **klipper-led_effect** by julianschill (`backups/2026-04-10/config/led_marcos.cfg`) |
| Full config file | `led_marcos.cfg` (filename is a typo — "marcos" instead of "macros") |

**Chain mapping**:

| Index | Role | Physical location |
|---|---|---|
| 1 – 26 | Main chamber lighting | Top strip under the frame top extrusions |
| 27 | "Logo" LED | Single accent, likely front badge area |
| 28 – 29 | Nozzle illumination | Two small LEDs on the toolhead |

**Effects defined**:

| Effect | Purpose | Triggered by |
|---|---|---|
| `rainbow` | Idle / ready | `leds_status_ready` macro + `autostart: true` |
| `bed_heating` | Animated during bed warm-up | `leds_bed_heating` |
| `bed_calibrating_z` | Breathing blue during `Z_TILT_ADJUST` | `leds_calibrate_z` |
| `logo` | Static purple on LED 27 | `leds_logo` |
| `nozzle` | White at the tool | `leds_nozzle` |
| `nozzle_bed_heating` | Breathing orange during heat-up | (called by `leds_bed_heating`) |
| `main_full` | Static white over LEDs 1–26 | `leds_printing` |

**Integration with `PRINT_START`** (`macros.cfg:50-79`): `leds_bed_heating` → `leds_calibrate_z` → `leds_printing` sequentially as the print proceeds through the bed heat → Z tilt → bed mesh → printing phases.

The file has commented-out placeholders for `tool_logo`, `tool_printing`, `tool_ready`, `critical_error`, and `comet` effects that were never finished.

---

## UI and remote access

### KlipperScreen touchscreen

| Spec | Value |
|---|---|
| Display | **Waveshare 5″ Capacitive 5-Point Touch Display** for Raspberry Pi 5 |
| Resolution | 800 × 480 |
| Interface | DSI (Pi 5 native DSI — confirmed by `rp1dsi` dmesg entries in `system_info.txt`) |
| Software | KlipperScreen v0.4.6-41-g5a5ae38 |
| Theme | material-darker, 300 s screen blanking |
| Config | `KlipperScreen.conf` — titlebar shows extruder, heater_bed, chamber_top |

### Mainsail (web UI)

- Bundled web UI served by nginx on port 80 of the Pi
- Mainsail installed at `~/mainsail` (built from release, no git — last updated Feb 16 2026)
- Includes file served from `/home/adam/mainsail-config/mainsail.cfg` (git repo, tracks upstream master)

### Moonraker + Moonraker-Obico

| Service | Version |
|---|---|
| Moonraker | v0.10.0-8 |
| Port | 7125 |
| Moonraker-Obico | v2.1.5 — remote AI monitoring / print-failure detection |
| Update manager | Tracks Klipper, Moonraker, Mainsail, KlipperScreen, crowsnest, sonar, klipper-led_effect, beacon_klipper (dev channel) |
| Not installed despite `moonraker.asvc` listing | MoonCord, moonraker-telegram-bot, octoeverywhere, ratos-configurator |

### Webcam

| Spec | Value |
|---|---|
| Camera | **Arducam 1080P 2MP IR-Cut Day/Night USB** (Microdia Vitade AF in the USB device ID) |
| Interface | USB 2.0 |
| Driver | crowsnest / ustreamer |
| Stream | `http://192.168.0.37/webcam/?action=stream`, 640×480 @ 15 fps |
| Snapshot | `http://192.168.0.37/webcam/?action=snapshot` |
| Special features | Auto IR-cut day/night switching, IR LEDs for low-light |
| Mounting | **TBD — confirm position and aim** (visible in photo at top of frame) |

---

## Filament path

| | |
|---|---|
| Feed | Single filament, 1.75 mm, direct-drive extruder |
| Upstream | **Sunlu filament dryer** — filament is pulled directly from the drier during printing, keeping it dry continuously |
| Tubing | PTFE guide tube from the dryer to the toolhead |
| Runout sensor | **None configured** (`[filament_switch_sensor]` lines are commented out throughout) |
| Filament change | `M600` macro defined (`macros.cfg:1-23`) — pauses, parks at `X130 Y0`, maintains temps, requires manual swap and `RESUME` |

No filament weight tracking or AMS-style multi-material support.

---

## Software stack

| Component | Version | Notes |
|---|---|---|
| Klipper | `v0.13.0-540-g57c2e0c96` (master, dirty) | Built 2026-02-14 |
| Moonraker | `v0.10.0-8` | Built 2026-02-08 |
| Mainsail | installed 2026-02-16 (no .git) | Stable channel via update_manager |
| mainsail-config | `v1.2.1-1` | — |
| KlipperScreen | `v0.4.6-41-g5a5ae38` | Built 2026-02-17 |
| Beacon | `v2.0.0-31-g4d2b15c` (dev channel) | Built 2026-03-19 |
| klipper-led_effect | `v0.0.18` | Built 2026-01-23 |
| crowsnest | `v4.1.17-1-g9cc3d4a` | Webcam daemon |
| sonar | `v0.2.0-1-g0d1d7c8` | Wi-Fi keep-alive (enabled but **not currently running** — worth checking) |
| moonraker-obico | `v2.1.5` | — |
| moonraker-timelapse | (Dec 2023 snapshot) | Installed but `[update_manager timelapse]` is commented out in `moonraker.conf` |
| kiauh | `v6.1.0` | Installer/manager |
| katapult | `v0.0.1-113-gec59b9b` | CAN bootloader — installed, not currently in use |

Full Python venv package lists captured in `backups/2026-04-10/system_info.txt` (4 venvs: `klippy-env`, `moonraker-env`, `.KlipperScreen-env`, `moonraker-obico-env`).

---

## Pin assignment reference

### Octopus Pro V1.0 — pins in use

| Pin | Purpose | Config reference |
|---|---|---|
| **Steppers (X/Y on TMC5160, Z×3 on TMC2209)** | | |
| `PF13/PF12/!PF14` | X step / dir / enable | `stepper_x` |
| `PG0/!PG1/!PF15` | Y step / dir / enable | `stepper_y` |
| `PF11/!PG3/!PG5` | Z step / dir / enable | `stepper_z` |
| `PG4/PC1/!PA0` | Z1 step / dir / enable | `stepper_z1` |
| `PC13/!PF0/!PF1` | Z2 step / dir / enable | `stepper_z2` |
| `PC4/PD11` | TMC5160 CS pins (X / Y) | `tmc5160 stepper_x/y` |
| `PA7/PA6/PA5` | SPI MOSI / MISO / SCLK (software, shared) | same |
| `PC6/PC7/PE4` | TMC2209 UART pins (Z / Z1 / Z2) | `tmc2209 stepper_z/z1/z2` |
| `!PG9` | Y sensorless diag0 | `tmc5160 stepper_y` |
| **Heaters and thermistors** | | |
| `PA1` | Bed heater (switches SSR → Honey Badger) | `heater_bed` |
| `PF3` | Bed thermistor (Generic 3950) | `heater_bed` |
| `PA3` | HotBox PTC heater (via MOSFET) | `heater_generic chamber` |
| `PF6` | Chamber top thermistor | `heater_generic chamber` |
| `PF5` | Chamber bottom thermistor | `temperature_sensor chamber_bottom` |
| `PF7` | HotBox element body thermistor | `temperature_fan hotbox` |
| **Fans** | | |
| `PD14` | HotBox 4028 fan pair | `temperature_fan hotbox` |
| `PD12` | Enclosure bay fans (PWM) | `output_pin enclosure_fans` |
| `PD13` | BentoBox fans (PWM, manual) | `output_pin bentobox` |
| `PD15` | Main 24 V lighting (PWM, default on) | `output_pin main_lights` |
| **Probe / Z home** | | |
| All Z axes | `probe:z_virtual_endstop` (Beacon) | `stepper_z/z1/z2` |
| `^PB7 / PB6` | BLTouch sensor / control (legacy, commented out) | — |
| **Lighting** | | |
| `PB0` | Neopixel chain data (29 LEDs, GRB) | `neopixel leds` |
| **Safety** | | |
| `PE11` | BTT safety relay control | `output_pin relay` |
| **Display / EXP** | | |
| `EXP1_1`..`EXP2_10` | Legacy ST7920 LCD (probably unused — superseded by KlipperScreen) | `board_pins`, `display` |
| `EXP1_1` | Beeper (`output_pin beeper`) | — |

### EBB36 Gen2 — pins in use (`ebb_gen2.cfg`)

| Pin | Purpose |
|---|---|
| `EBB36:PB14/PA8/!PB6` | Extruder step / dir / enable |
| `EBB36:PB3` | Extruder TMC2209 UART |
| `EBB36:PB4` | Hotend heater output |
| `EBB36:PA3` | Hotend PT1000 thermistor (with 2200 Ω pullup) |
| `EBB36:PD3` | Part-cooling fan |
| `EBB36:PA5` | Heat-break fan (on > 50 °C extruder) |
| `EBB36:^!PA15` | X endstop switch (toolhead-mounted) |
| `EBB36:PA0` | Driver temperature thermistor (EBB36_Driver) |
| `EBB36:PB1 / spi2_PB2_PB11_PB10` | Onboard LIS2DW accelerometer — **not used** (commented out) |

---

## Bill of materials

### Electronics

| # | Part | Model | Qty | Source |
|---|---|---|---|---|
| 1 | Main controller | BigTreeTech Octopus Pro V1.0 (STM32F429ZGT6) | 1 | [Amazon UK B09JC2NR1L](https://www.amazon.co.uk/dp/B09JC2NR1L) |
| 2 | Toolhead board | BigTreeTech EBB36 Gen2 V1.0 | 1 | BTT / resellers |
| 3 | Single-board computer | Raspberry Pi 5 (8 GB) | 1 | — |
| 4 | X/Y stepper drivers | BigTreeTech TMC5160T Pro V1.0 SPI stepstick with heatsink | 2 | BTT / resellers |
| 5 | Z stepper drivers | BigTreeTech TMC2209 V1.3 UART stepstick | 3 | BTT / resellers |
| 6 | Safety relay | BigTreeTech Relay V1.2 Power Monitoring Module (incl. 10 A / 250 V rocker switch) | 1 | BTT / resellers |
| 7 | MOSFET module | Heat-bed MOSFET expansion, 25 A MOS tube | 2 | Generic 3D-printer accessory |
| 8 | Solid-state relay | Heschen SSR-40DA (4–32 VDC / 24–480 VAC / 40 A) | 1 | — |
| 9 | Touchscreen | Waveshare 5″ Capacitive 5-Point Touch Display (800×480, DSI) | 1 | — |
| 10 | Webcam | Arducam 1080P 2 MP IR-Cut USB (day/night) | 1 | — |
| 11 | PSU | Stock Ender 5 Pro PSU (~24 V 350 W Meanwell-equivalent) | 1 | — |

### Motion

| # | Part | Model | Qty |
|---|---|---|---|
| 12 | X/Y stepper motors | **LDO-42STH48-2804AC** "Super Power" (2.8 A, 0.55 Nm) | 2 |
| 13 | Z stepper motors | **Wotrees NEMA 17 17HS4401** (1.5 A, 0.42 Nm) | 3 |
| 14 | Extruder stepper | **LDO pancake** (shipped with Orbiter 2.5) | 1 |
| 15 | Z lead screws | Sourcingmap Tr8×2 stainless, 350 mm, 2 mm pitch / 8 mm lead, with brass nut | 3 |
| 16 | Z couplers | Oldham | 3 |
| 17 | Belts | GT2 6 mm (dual CoreXY, 4-anchor ZeroG style) | 4 loops |
| 18 | X/Y linear rails | MGN12H (TBD — confirm length from Nebula Pro kit) | 3 |

### Toolhead

| # | Part | Model | Qty |
|---|---|---|---|
| 19 | Toolhead body | VzBot CNC toolhead — **Zero G edition** (aluminium) | 1 |
| 20 | Hotend | Phaetus Rapido 2 HF (High Flow, 24 V, 90 W planar ceramic, black) | 1 |
| 21 | Nozzle | Phaetus hardened steel (standard) | 1 |
| 22 | Silicone sock | Phaetus standard | 1 |
| 23 | Extruder | LDO **Orbiter 2.5** with stock pancake stepper | 1 |
| 24 | Hotend thermistor | PT1000 (integrated in Rapido 2) | 1 |
| 25 | Probe | **Beacon RevH** eddy-current contactless | 1 |
| 26 | Probe duct | Tentacool / Goliath Short (printed ABS or ASA) | 1 |
| 27 | Part-cooling fans | 24 V (TBD size — 4010 or 5015 blower — **confirm**) | — |
| 28 | Heat-break fan | 24 V (TBD — **confirm**) | 1 |

### Bed and frame

| # | Part | Model | Qty |
|---|---|---|---|
| 29 | Frame kit | ZeroG **Nebula Pro** (LDO Motors) — pre-cut 2020/2040 anodised extrusions | 1 set |
| 30 | Enclosure panels | 3 mm transparent acrylic | set |
| 31 | Bed module | ZeroG Hydra 275 (Ender 5 Pro variant) | 1 |
| 32 | Bed plate | 10 mm solid aluminium, 275 × 275 | 1 |
| 33 | Bed surface | PEI magnetic stick-on sheet | 1+ (swappable) |
| 34 | Bed heater | Fabreeko **Honey Badger** 600 W 110/220 V AC silicone | 1 |
| 35 | Bed thermistor | Generic 3950 | 1 |
| 36 | Kinematic mount | Kossel balls on aluminium bed arms | 3 |
| 37 | Aluminium upgrades | CNC aluminium hotend mount, bed arms, stepper motor mounts | set |

### Chamber conditioning (HotBox)

Per [github.com/nark3d/HotBox](https://github.com/nark3d/HotBox) BOM, with the 4028 fan upgrade applied:

| # | Part | Spec | Qty | Source |
|---|---|---|---|---|
| 38 | PTC heater element | 24 V 200 W ceramic, self-limiting | 1 | [Amazon UK B0DK5W3RKS](https://www.amazon.co.uk/dp/B0DK5W3RKS) |
| 39 | Thermal fuse | 125 °C square ceramic, 250 V / 15 A, CQC rated | 1 | [Amazon UK B0BNQ7SJYP](https://www.amazon.co.uk/dp/B0BNQ7SJYP) |
| 40 | Axial fans | 24 V **4028** (upgraded from 4020) | 2 | — |
| 41 | Thermistor | NTC 100 K, OD 3 mm | 1 | [Amazon UK B0BDFHZZQX](https://www.amazon.co.uk/dp/B0BDFHZZQX) |
| 42 | Thermal adhesive | EC360 2 W/mK | 1 | [Amazon UK B00XQ9AZ8Y](https://www.amazon.co.uk/dp/B00XQ9AZ8Y) |
| 43 | Heat-set inserts | M3 × 4 mm brass | 8 | [Amazon UK B0DC6RXRB9](https://www.amazon.co.uk/dp/B0DC6RXRB9) |

### Chamber conditioning (BentoBox)

| # | Part | Spec | Qty |
|---|---|---|---|
| 44 | Printed housing | Standard Voron BentoBox (community print) | 1 |
| 45 | Fans | 24 V 4028 axial | 2 |
| 46 | Filter media | Activated carbon | 1 fill |

### Lighting

| # | Part | Spec | Qty |
|---|---|---|---|
| 47 | Neopixel chain | WS2812-family, 29 LEDs, GRB, 24 V (**TBD**: confirm if chain is 5 V strip with level shift, or 24 V variant) | 1 strip |

### Filament handling

| # | Part | Spec | Qty |
|---|---|---|---|
| 48 | Filament dryer | Sunlu (model **TBD** — confirm S1 / S2 / E2) | 1 |
| 49 | Filament guide tube | PTFE | 1 |

---

## Known gaps and future work

1. **CAN bus conversion incomplete.** Katapult bootloader is flashed and the EBB36 has CAN-capable firmware, but the Pi does not have `can0` configured and the EBB is running over USB via the main umbilical. The `config-backup-pre-canbus.tar` from 2026-03-22 in `backups/2026-04-10/misc/` is the pre-attempt snapshot. Finishing this would free the USB port, simplify the umbilical, and enable higher-reliability toolhead comms.

2. **`sonar` service enabled but not running.** The Wi-Fi keep-alive daemon is installed and enabled but was not in the running-services list at backup time. If Wi-Fi drops are a problem, worth investigating why it isn't running.

3. **Moonraker-Timelapse half-wired.** `timelapse.cfg` is included from Klipper via a symlink, but the corresponding `[update_manager timelapse]` is commented out in `moonraker.conf`. Either fully enable or fully remove.

4. **No safety interlocks.** No door switch, no exhaust fan for the enclosure (for fume management when printing ASA/ABS), no emergency stop beyond the rocker switch on the BTT safety relay. Consider adding at minimum a door interlock for when the HotBox is running at 70 °C.

5. **BentoBox has no control macro.** It's only toggleable by setting `SET_PIN PIN=bentobox VALUE=1.0` manually in Mainsail. A `FILTER_ON` / `FILTER_OFF` macro pair (and automatic engagement during ASA prints via `PRINT_START`) would be an obvious next win.

6. **HotBox thermistor is NTC 100K** (per GitHub BOM), but the Klipper config declares `sensor_type: Generic 3950` in `temperature_fan hotbox` — check whether this is a deliberate choice (3950 beta matches many no-name 100K thermistors) or if it needs recalibration against the specific thermistor part used.

7. **Orca slicer accel mismatch.** Orca's printer profile has `machine_max_acceleration_x/y: 40000`, but Klipper caps at 10000. The Orca value is effectively aspirational — the Klipper ceiling wins silently. Harmless, just stale.

8. **TBD hardware details that should be confirmed and filled in later**:
   - Overall outer frame dimensions (not yet measured)
   - Exact X/Y linear rail model and length (confirm against Nebula Pro kit)
   - Exact count and model of enclosure-bay cooling fans (`PD12`)
   - Exact routing of the second 25 A MOSFET module (drives what?)
   - Webcam mounting position and aim
   - Sunlu filament dryer exact model
   - Neopixel chain voltage (5 V addressable with level shifter vs 24 V variant)
   - Toolhead part-cooling and heat-break fan exact models

---

*End of hardware reference. Updates: edit this file in place and re-commit. For a full machine-identity capture (installed services, git commits, pip freezes, kernel version, etc.) see `backups/2026-04-10/system_info.txt`.*
