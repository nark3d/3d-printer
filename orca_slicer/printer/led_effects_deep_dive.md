# Klipper LED Effects — Deep Dive Research

> Research document for redesigning the LED macro system on the ZeroG Mercury One.  
> **Current hardware**: 24× WS2812B NeoPixels (12 per side of bed) on Octopus Pro PB0, GRB colour order.  
> **Possible future**: NeoPixel(s) on EBB36 Gen2 toolhead board RGB header (pin TBD — PD3 is currently allocated to part cooling fan; actual RGB header pin on Gen2 V1.0 needs verification from BTT schematic or board silkscreen).  
> **No toolhead LEDs currently** — VzBot CNC aluminium toolhead has no mounting adapter for NeoPixels.

---

## Table of contents

1. [Current setup analysis](#1-current-setup-analysis)
2. [Klipper native LED support](#2-klipper-native-led-support)
3. [klipper-led_effect plugin — full reference](#3-klipper-led_effect-plugin--full-reference)
4. [What other builders use — community configs](#4-what-other-builders-use--community-configs)
5. [EBB36 Gen2 RGB output](#5-ebb36-gen2-rgb-output)
6. [Where to trigger LED state changes](#6-where-to-trigger-led-state-changes)
7. [Colour science — what the bed LEDs should do during printing](#7-colour-science--what-the-bed-leds-should-do-during-printing)
8. [Recommended effect design per printer state](#8-recommended-effect-design-per-printer-state)
9. [Cool multi-layer recipes from the community](#9-cool-multi-layer-recipes-from-the-community)
10. [Performance and gotchas](#10-performance-and-gotchas)
11. [Implementation roadmap](#11-implementation-roadmap)
12. [References](#12-references)

---

## 1. Current setup analysis

### What's physically connected

| Component | Detail |
|---|---|
| LED strip | 24× WS2812B NeoPixels, GRB colour order |
| Layout | 12 per side of bed (left + right) |
| Data pin | `PB0` on Octopus Pro (note: some Octopus Pro revisions have a silkscreen error where PB0 is electrically PB10 [31] — ours works with `PB0` so not an issue) |
| Toolhead LEDs | **None currently** — the VzBot CNC aluminium toolhead doesn't have a standard NeoPixel mount adapter |
| EBB36 RGB header | Available but the exact Gen2 V1.0 pin needs checking (PD3 is used by part cooling fan; the RGB header is a separate physical connector with its own pin — check BTT EBB36 Gen2 schematic or board silkscreen) |

### What's in the config (`led_marcos.cfg`)

| Item | Status | Issue |
|---|---|---|
| `chain_count: 29` | **Wrong** — only 24 LEDs physically connected | LEDs 25-29 are ghost addresses (no hardware) |
| LED 27 = "logo" | **Doesn't exist** — was on old toolhead | Effects targeting `(27)` do nothing |
| LEDs 28-29 = "nozzle" | **Don't exist** — were on old toolhead | Effects targeting `(28,29)` do nothing |
| `bed_heating` range `(1-27)` | **Should be `(1-24)`** | Includes 3 ghost LEDs |
| `bed_calibrating_z` range `(1-26)` | **Should be `(1-24)`** | Includes 2 ghost LEDs |
| `main_full` range `(1-26)` | **Should be `(1-24)`** | Includes 2 ghost LEDs |
| `leds_logo` macro | **No effect** — logo LED doesn't exist | Dead macro |
| `leds_nozzle` macro | **No effect** — nozzle LEDs don't exist | Dead macro |
| `leds_printing` calls `leds_logo` + `nozzle` | **Partially dead** — only `main_full` actually does anything | 2 of 3 sub-calls are no-ops |
| ~60 lines of commented-out code | Dead weight | References old `neopixel: lights` name |
| Filename `led_marcos.cfg` | Typo | Should be `led_macros.cfg` (requires `[include]` update in printer.cfg) |

### What currently works in practice

The only visible effects are on the 24 bed LEDs:
- **Startup**: rainbow gradient cycles across all 24 LEDs (autostart)
- **Bed heating**: blue-to-orange heater-reactive effect on 24 LEDs + orange breathing on ghost nozzle LEDs (invisible)
- **Z calibration**: blue breathing on 24 LEDs
- **Printing**: static white on 24 LEDs
- **Print end**: back to rainbow

The logo and nozzle effects fire but target nonexistent LEDs. No errors — Klipper silently ignores commands to chain indices beyond the physical count, but it's wasted processing.

### Two logical LED groups for effects

With 12 LEDs per side, the chain can be split into two mirrored halves for symmetrical effects:

```
Left side:  neopixel:leds (1-12)
Right side: neopixel:leds (13-24)     — or reversed: (24-13) for mirrored animations
```

This is a powerful feature of klipper-led_effect — one effect block can reference both ranges, and using `(24-13)` reverses the direction so a `comet` or `chase` effect sweeps inward from both sides simultaneously [14][15].

---

## 2. Klipper native LED support

### Supported hardware types [1][2]

| Config section | Hardware | Data protocol |
|---|---|---|
| `[neopixel]` | WS2812 / WS2812B / SK6812 | Single-wire, GRB or RGBW |
| `[dotstar]` | APA102 | Two-wire (clock + data) |
| `[led]` | Generic single-colour PWM | Direct GPIO PWM |
| `[pca9533]` / `[pca9632]` | I2C PWM controllers | I2C bus |

### Native gcode commands [6]

**`SET_LED`** — direct LED control:
```
SET_LED LED=<name> RED=<0-1> GREEN=<0-1> BLUE=<0-1> [WHITE=<0-1>] [INDEX=<n>] [TRANSMIT=0] [SYNC=1]
```
- `INDEX` targets a single LED (1-based)
- `TRANSMIT=0` batches updates — set each LED individually, then send one final command without this flag to push all changes at once
- `SYNC=1` (default) synchronises with the gcode queue

**`SET_LED_TEMPLATE`** — continuous template-driven updates [7]:
```
SET_LED_TEMPLATE LED=<name> TEMPLATE=<template_name> [INDEX=<n>]
```
- Assigns a `[display_template]` that Klipper continuously re-evaluates
- Template produces comma-separated RGBA values (0.0–1.0)
- Can read `printer.virtual_sdcard.progress` for progress bars without the led_effect plugin
- The `digitalninja-ro/klipper-neopixel` repo [8][9] uses this approach for progress bars, temperature displays, and speed displays — no plugin required

### Chain limits

Historical cap was 85 LEDs. PR #3380 [5] raised the limit to ~166 RGB or ~125 RGBW LEDs per chain. 24 LEDs is well within limits.

---

## 3. klipper-led_effect plugin — full reference

**Repository**: https://github.com/julianschill/klipper-led_effect [10]  
**Documentation**: https://github.com/julianschill/klipper-led_effect/blob/master/docs/LED_Effect.md [11]  
**DeepWiki**: https://deepwiki.com/julianschill/klipper-led_effect [12]

Already installed on this printer via the existing `led_marcos.cfg`. Managed via Moonraker's update_manager (check `moonraker.conf` for the `[update_manager led_effect]` section).

### 3.1 Effect block structure

```ini
[led_effect my_effect]
leds:
    neopixel:leds              # whole chain
    neopixel:leds (1-12)       # index range
    neopixel:leds (12-1)       # reversed range (for mirrored effects)
    neopixel:leds (1,5,9)      # specific indices
autostart:    false            # start on Klipper boot?
frame_rate:   24               # FPS
run_on_error: false            # activate on MCU shutdown/error?
recalculate:  false            # re-evaluate layers on each activation?
heater:       heater_bed       # required for heater-reactive layers
stepper:      x                # required for stepper-reactive layers
analog_pin:   PA1              # required for analog-reactive layers
endstops:     x, y, z, probe   # required for homing-reactive layers
layers:
    <type> <rate> <cutoff> <blend> <(r,g,b),...>
```

Multiple LED strips can be combined in one effect — including mixing `neopixel:` and `dotstar:` in the same effect block [14].

### 3.2 Complete layer type reference

#### Static / pattern layers

| Layer | Rate | Cutoff | Description |
|---|---|---|---|
| `static` | 0 | 0 | Palette colours blended evenly across the strip. No animation. |
| `pattern` | time between shifts | shift distance (LED positions) | Repeating pattern that shifts across the strip over time. |

#### Cycling / animation layers

| Layer | Rate | Cutoff | Description |
|---|---|---|---|
| `linearfade` | cycle duration | — | Cycles through palette colours in order with linear fade |
| `breathing` | cycle duration | — | Sinusoidal brightness wave through palette colours |
| `blink` | cycle duration | on-time ratio (0.0–1.0) | Alternating on/off; cutoff controls duty cycle |
| `strobe` | flashes/second | decay rate (higher = faster) | Rapid strobe flash; colours cycle per flash |
| `twinkle` | probability (0=never, 254=always) | decay rate | Random LEDs light with random palette colours. Rendered per-frame (not pre-rendered) [13] |
| `gradient` | cycle speed (negative = reverse) | number of gradients | Linear gradient cycles across strip |
| `comet` | speed (negative = reverse) | tail length | Single comet head + gradient tail |
| `chase` | speed (negative = reverse) | tail length | Multiple comets chasing each other |
| `cylon` | sweep speed | — | Larson Scanner / Knight Rider — light bounces between ends [17] |

#### Heater / temperature reactive layers

| Layer | Rate | Cutoff | Requires | Description |
|---|---|---|---|---|
| `heater` | min activation temp | 0 or 1 (disable at target) | `heater:` | Gradient tracks heater progress toward target |
| `temperature` | cold threshold | hot threshold | `heater:` | Maps absolute temperature to palette gradient |
| `heatergauge` | trailing LEDs | leading LEDs | `heater:` | Bar gauge fills as temperature rises |
| `temperaturegauge` | cold temp | hot temp | `heater:` | Bar gauge with explicit temp range |
| `fire` | spark probability (0-255) | cooling rate | — | FastLED Fire2012 port. Rendered per-frame [13] |
| `heaterfire` | min activation temp | 0 or 1 (disable at target) | `heater:` | Fire that scales with heater temperature — ember at cold, blaze at target |

#### Motion / progress reactive layers

| Layer | Rate | Cutoff | Requires | Description |
|---|---|---|---|---|
| `stepper` | trailing LEDs | leading LEDs | `stepper:` | Bar tracks stepper position (0-100%) |
| `steppercolor` | position scaling | position offset | `stepper:` | Maps stepper position to palette colour |
| `progress` | trailing LEDs | leading LEDs | — | Bar tracks print progress percentage |

#### Endstop / button reactive layers

| Layer | Rate | Cutoff | Requires | Description |
|---|---|---|---|---|
| `homing` | decay rate (higher = slower) | — | `endstops:` | Flash + fade when endstop triggers during homing. Automatic — no macro needed |
| `switchbutton` | fade-in time | fade-out time | `button_pins:` | On when pressed, off when released |
| `togglebutton` | fade-in time | fade-out time | `button_pins:` | Press toggles on/off |
| `flashbutton` | flash duration | — | `button_pins:` | Brief flash per press |

#### Analog input layer

| Layer | Rate | Cutoff | Requires | Description |
|---|---|---|---|---|
| `analogpin` | signal multiplier | min trigger threshold | `analog_pin:` | Maps ADC voltage to palette colour |

### 3.3 Blend modes [18][19]

All 14 available blend modes, evaluated bottom-up:

| Mode | Effect |
|---|---|
| `bottom` | Base layer (blend mode ignored on lowest layer) |
| `top` | Completely replaces layers below |
| `add` | Colours added — result brighter |
| `subtract` | Top subtracted from bottom — similar colours darken |
| `subtract_b` | Bottom subtracted from top |
| `difference` | Absolute difference |
| `average` | Simple average |
| `multiply` | Channels multiplied — result darker |
| `divide` | Bottom divided by top — brighter |
| `divide_inv` | Top divided by bottom |
| `screen` | Invert + multiply + invert — smoother version of add |
| `lighten` | Per-channel max |
| `darken` | Per-channel min |
| `overlay` | Multiply (dark areas) + screen (light areas) |

### 3.4 Gcode commands [20][21]

| Command | Description |
|---|---|
| `SET_LED_EFFECT EFFECT=name` | Activate effect |
| `SET_LED_EFFECT EFFECT=name STOP=1` | Deactivate effect |
| `SET_LED_EFFECT EFFECT=name REPLACE=1` | Stop all other effects on those LEDs, then start this one |
| `SET_LED_EFFECT EFFECT=name FADETIME=2.0` | Fade in over 2 seconds |
| `SET_LED_EFFECT EFFECT=name STOP=1 FADETIME=1.0` | Fade out over 1 second |
| `SET_LED_EFFECT EFFECT=name RESTART=1` | Restart from beginning |
| `STOP_LED_EFFECTS` | Stop all effects globally |
| `STOP_LED_EFFECTS LEDS="neopixel:leds"` | Stop all effects on a specific strip |
| `STOP_LED_EFFECTS LEDS="neopixel:leds (1-12)"` | Stop on specific range |

**Crossfade** — `REPLACE=1` + `FADETIME` gives a smooth transition between states:
```
SET_LED_EFFECT EFFECT=printing REPLACE=1 FADETIME=1.0
```

### 3.5 Key configuration options

| Option | When to use |
|---|---|
| `autostart: true` | Idle/standby animation that should run from boot |
| `run_on_error: true` | Error indicator — activates on MCU shutdown/thermal runaway. Cannot be triggered by macros since klippy is shutting down [22] |
| `recalculate: true` | Re-evaluates layer parameters on each activation — enables parameterised effects [23] |
| `heater:` | Required for `heater`, `temperature`, `heatergauge`, `temperaturegauge`, `heaterfire` layers |
| `stepper:` | Required for `stepper`, `steppercolor` layers. Values: `x`, `y`, `z` |
| `endstops:` | Required for `homing` layer. Values: `x`, `y`, `z`, `probe` |

---

## 4. What other builders use — community configs

### 4.1 Voron Stealthburner — the gold standard [32][33][34]

The official Voron Stealthburner config (`stealthburner_leds.cfg`) defines these states using simple `SET_LED` calls:

| Macro | Logo | Nozzle | Purpose |
|---|---|---|---|
| `STATUS_OFF` | Off | Off | Everything off |
| `STATUS_READY` | White fade | White | Idle |
| `STATUS_BUSY` | Blue | White | Processing |
| `STATUS_HEATING` | Pulsing red/orange | Off | Heat soak |
| `STATUS_LEVELING` | Green | White | Z tilt / QGL |
| `STATUS_HOMING` | Cyan | White | Homing |
| `STATUS_CLEANING` | Green | Full white | Nozzle brush |
| `STATUS_MESHING` | Purple | White | Bed mesh |
| `STATUS_CALIBRATING_Z` | Orange | White | Z offset calibration |

**With klipper-led_effect** — the "Rainbow BARF" configs [35][36][37][38] replace simple SET_LED calls with animated effects: rainbow gradient on the logo PCB, breathing on nozzle, heater-reactive colours during warm-up.

### 4.2 Klippain by Frix-x [39]

Full macro framework where LED states follow the printer state machine automatically — no manual macro calls needed. Status transitions drive LED changes.

### 4.3 VzBot configurations [45][46][47][48]

- **VzLight LED PCB** [45] — Vector 3D's official addressable LED PCB for VzBot frames
- Community configs use neopixel effects for case lighting with heater-reactive bed warming effects

### 4.4 Mercury One / ZeroG [49][50]

- **Braden5790/ZeroG_Mercury_One.1_Hydra** [49] — custom config with LED integration
- **ethomasgt/Ender-5-Plus-Mercury-One** [50] — 35-LED chain config on SKR3

### 4.5 Progress bar implementations

- **digitalninja-ro/klipper-neopixel** [8][9] — progress bars using `SET_LED_TEMPLATE` (no plugin needed). Macros: `NEOPIXEL_DISPLAY LED="leds" TYPE=print_percent MODE=progress`
- **Printables LED progress bar mount** by Kotvic [51] — physical mount with matching klipper-led_effect config
- **klipper-led_effect `progress` layer** [11] — built-in progress tracking, fills a bar gauge as print % increases

### 4.6 WLED integration (WiFi-controlled, ESP32) [61][68][69]

For complex standalone animations beyond what klipper-led_effect can do. Moonraker has native WLED support. Overkill for 24 LEDs but powerful for very long runs or pixel-art-style displays.

### 4.7 Notable community repos

| Repo | Builder | Features |
|---|---|---|
| rootiest/zippy-klipper_config [54][55] | Voron | Full BARF integration, comprehensive macros |
| xkonni/klipper_config [56] | Voron V2.4 | Frame + toolhead NeoPixels |
| Laker87/klipper_config [43] | Octopus | Case light + heater-reactive effects |
| bistory Gist [44] | Generic | Well-commented effect examples |
| jschuh/klipper-macros [66] | Generic | `GCODE_ON_PRINT_STATUS` event system |
| Happy Hare MMU [52][53] | MMU | Gate-status LEDs, virtual LED chains |

---

## 5. EBB36 Gen2 RGB output

### What the documentation says [29][30]

The EBB36 Gen2 V1.0 has a dedicated RGB header for NeoPixel/WS2812 LEDs:

- **Output**: 5V level-shifted digital signal — fully NeoPixel compatible
- **Protection**: ESD protection + Schottky diode for short-circuit protection
- **Chain**: can drive small chains (1–10+ LEDs, limited by 5V current draw from the toolhead board)
- **Colour order**: GRB (WS2812B) or GRBW (SK6812)

### Current pin situation on our EBB36

| Pin | Currently used for | Available for LEDs? |
|---|---|---|
| PD3 | Part cooling fan (FAN0) | **No** — in use |
| PA5 | Heatbreak cooling fan (FAN1) | **No** — in use |
| RGB header pin | **Unknown** — needs checking on the physical board | **Possibly** — the Gen2 has a dedicated RGB header separate from the fan headers |

**Action needed**: check the BTT EBB36 Gen2 V1.0 schematic or the board silkscreen to identify the actual RGB header pin. The Gen2 documentation [29] suggests it should be separate from PD3. Common candidates on BTT toolhead boards: `PB7`, `PD2`, or `PA2`.

### If toolhead LEDs are added later

Config would be:
```ini
[neopixel toolhead_leds]
pin: EBB36:<RGB_PIN>       # actual pin from schematic
chain_count: 1-3            # typical for a small nozzle/logo LED
color_order: GRB            # or GRBW for SK6812
```

Effects would target `neopixel:toolhead_leds` alongside the existing `neopixel:leds` bed chain.

---

## 6. Where to trigger LED state changes

### 6.1 PRINT_START macro [62]

The primary integration point. Match LED states to each phase:

```gcode
[gcode_macro PRINT_START]
gcode:
    SET_LED_EFFECT EFFECT=heating REPLACE=1 FADETIME=1.0    # bed+extruder warming
    M140 S{BED_TEMP}
    M104 S{EXTRUDER_TEMP}
    G28
    SET_LED_EFFECT EFFECT=homing REPLACE=1 FADETIME=0.5     # homing
    Z_TILT_ADJUST
    SET_LED_EFFECT EFFECT=meshing REPLACE=1 FADETIME=0.5    # bed mesh
    BED_MESH_CALIBRATE ADAPTIVE=1
    M190 S{BED_TEMP}
    SMART_PARK
    M109 S{EXTRUDER_TEMP}
    SET_LED_EFFECT EFFECT=printing REPLACE=1 FADETIME=1.0   # printing
    LINE_PURGE
```

### 6.2 PRINT_END macro [62]

```gcode
[gcode_macro PRINT_END]
gcode:
    # ... retract, park ...
    SET_LED_EFFECT EFFECT=complete REPLACE=1 FADETIME=2.0
    UPDATE_DELAYED_GCODE ID=return_to_idle DURATION=60       # back to idle after 60s
```

### 6.3 Idle / startup via `delayed_gcode` [62][63]

```gcode
[delayed_gcode startup_leds]
initial_duration: 1
gcode:
    SET_LED_EFFECT EFFECT=idle FADETIME=2.0

[delayed_gcode return_to_idle]
gcode:
    SET_LED_EFFECT EFFECT=idle REPLACE=1 FADETIME=3.0
```

Or simply use `autostart: true` on the idle effect — `REPLACE=1` in other macros will displace it, and when all effects stop, autostart effects resume.

### 6.4 PAUSE / RESUME / M600 [64][65]

```gcode
[gcode_macro PAUSE]
rename_existing: BASE_PAUSE
gcode:
    BASE_PAUSE
    SET_LED_EFFECT EFFECT=paused REPLACE=1 FADETIME=1.0

[gcode_macro RESUME]
rename_existing: BASE_RESUME
gcode:
    SET_LED_EFFECT EFFECT=printing REPLACE=1 FADETIME=0.5
    BASE_RESUME
```

### 6.5 Automatic homing detection (no macro needed) [11]

The `homing` layer type triggers automatically when the specified endstops fire:

```ini
[led_effect homing_flash]
autostart: true
endstops: x, y, z, probe
leds:
    neopixel:leds
layers:
    homing 2.5 0 add (0.3, 0.3, 1.0)
```

### 6.6 Error / MCU shutdown (automatic, no macro possible) [22]

```ini
[led_effect critical_error]
run_on_error: true
leds:
    neopixel:leds
layers:
    strobe    1  1.5  add        (1.0, 1.0, 1.0)
    breathing 2  0    difference (0.95, 0.0, 0.0)
    static    1  0    top        (1.0, 0.0, 0.0)
```

This fires when Klipper enters shutdown state (thermal runaway, MCU disconnect, etc.). It cannot be called from macros because klippy is shutting down when it activates.

### 6.7 Event-driven via jschuh/klipper-macros [66]

```gcode
GCODE_ON_PRINT_STATUS STATUS=printing COMMAND="SET_LED_EFFECT EFFECT=printing REPLACE=1"
GCODE_ON_PRINT_STATUS STATUS=complete COMMAND="SET_LED_EFFECT EFFECT=complete REPLACE=1"
```

---

## 7. Colour science — what the bed LEDs should do during printing

### Why full white is wrong for bed NeoPixels

WS2812B "full white" (`R=1, G=1, B=1`) is three narrow-band colour peaks (620nm red, 525nm green, 465nm blue) with spectral gaps between them. CRI is ~60-70 — colours look washed out with a purple/blue cast. Compare with the dedicated white LED strips in the enclosure which use phosphor-coated LEDs with broad-spectrum output (CRI 80-95+).

**The enclosure strips are the primary illumination. The bed NeoPixels have a different job.**

### The bed LEDs' real advantage: low-angle side lighting

Mounted at bed level, the NeoPixels shine across the print surface at a shallow angle. This is the ideal geometry for **highlighting surface defects** — layer lines, seams, zits, stringing, warping. Overhead light (from the enclosure strips) washes these out because it hits the surface head-on. Low-angle side light creates shadows in every tiny imperfection.

### Recommended colour during printing

**Warm/neutral accent, moderate brightness** — not full blast competing with the overhead strips:

| Setting | Value | Rationale |
|---|---|---|
| Colour | `(0.8, 0.6, 0.3)` warm amber | Avoids the WS2812B purple cast; warm tone complements cooler overhead white for natural depth |
| Brightness | ~60-80% of max | Enough for side-shadow enhancement without overpowering enclosure lights |
| Pattern | `static` (no animation) | Stable light for print inspection — no distracting movement |

Alternatives:
- `(0.7, 0.7, 0.5)` — more neutral, less warm
- `(0.5, 0.5, 0.5)` — half-brightness white if you prefer colour-neutral
- Complementary-to-filament colour for maximum defect contrast (e.g., magenta accent for green filament) — impractical for everyday use but powerful for quality inspection sessions

---

## 8. Recommended effect design per printer state

Based on community consensus, the Voron/VzBot standard patterns [32][35][44][54], and the colour science above:

### For 24 bed LEDs (12 per side), no toolhead LEDs

**Primary illumination from enclosure LED strips. Bed NeoPixels are accent/status/side-lighting.**

| State | Effect type | Colours | Description | Why |
|---|---|---|---|---|
| **Idle / standby** | `breathing` or slow `gradient` | Soft blue or rainbow | Gentle animation showing printer is alive | Community standard — rainbow is the most popular idle [35][54] |
| **Heating (bed)** | `heaterfire` with mirrored 12+12 | Orange/red ember → blaze | Scales with bed temp — comets sweep inward from both edges toward centre | Voron standard — visually intuitive. Mirrored layout matches physical heat spread [35][37] |
| **Heating (extruder)** | `heater` | Blue → orange → white | Cooler palette than bed to differentiate | Distinguishes which heater is active |
| **Heating (chamber)** | `temperature` | Blue → amber | Maps chamber temp to colour gradient | Immediately relevant for ASA/ABS — tells you chamber soak state at a glance |
| **Homing** | Per-axis `homing` layers | Red=X, Green=Y, Blue=Z/probe | Auto-triggers per endstop, different colour per axis | Functional: tells you which axis just homed. No macro needed [11] |
| **Z tilt / levelling** | `breathing` | Purple/magenta | Slow breathing indicates calibration in progress | Voron STATUS_MESHING uses purple [32] |
| **Bed meshing** | `chase` or `comet` mirrored | Purple/blue sweep inward | Moving effect suggests scanning motion — sweeps from both sides | More dynamic than breathing; mirrored layout is visually striking |
| **Printing** | `static` warm accent | `(0.8, 0.6, 0.3)` | Warm side-lighting for surface defect visibility | See §7 — enclosure strips are primary illumination; bed LEDs provide low-angle shadow enhancement |
| **First layers (1-3)** | `static` full bright | `(1.0, 0.9, 0.7)` near-white | Maximum visibility during the critical first layers | You're most likely watching during first layer; dim to warm accent after layer 3 |
| **Paused / M600** | `breathing` | Yellow/amber | Pulsing attention-getter — "I need you" | Yellow = attention across all UI conventions |
| **Print complete** | `breathing` | Green | "Success" — gentle animation | Green = done/success [32] |
| **Cool-down (safe to touch)** | `temperature` | Green → blue | Tracks bed temp: green at print completion, shifts to blue as bed cools below ~40°C | Functional: tells you at a glance from across the room whether it's safe to grab the print |
| **Layer change heartbeat** | Brief brightness pulse | Current colour + white flash | Very subtle quick pulse on each layer change | Visual "heartbeat" — if pulses stop, something's wrong. Visible at a glance without checking Mainsail |
| **Error / shutdown** | `strobe` + `breathing` | Red + white flash | Urgent alarm pattern | **NOTE: effectively useless with current relay setup** — relay cuts power within milliseconds of shutdown, LEDs go dark. Would only work if LEDs were powered independently from the relay circuit |
| **Lights off** | — | All off | Maintenance / sleeping | `SET_LED RED=0 GREEN=0 BLUE=0` |

### Mirrored effects for 12+12 layout

For effects like `comet` and `chase`, using reversed ranges creates a symmetrical sweep inward from both sides:

```ini
leds:
    neopixel:leds (1-12)
    neopixel:leds (24-13)      # reversed — comet sweeps inward from right side too
```

This makes `comet` and `chase` sweep from the outer edges toward the centre simultaneously — much more visually striking than a single sweep across all 24 [14][15].

---

## 9. Cool multi-layer recipes from the community

### 8.1 Rainbow idle with brightness pulse [35]

```ini
layers:
    gradient  0.3  1  top      (1.0,0.0,0.0),(0.0,1.0,0.0),(0.0,0.0,1.0)
    breathing 8    0  multiply (0.5, 0.5, 0.5)
```
Rainbow gradient with a slow 8-second brightness pulse. The `multiply` blend dims the gradient during the breathing trough.

### 8.2 Heater fire (bed warming) [44]

```ini
heater: heater_bed
layers:
    heaterfire 40 0 add (0.0,0.0,0.0),(1.0,0.2,0.0),(1.0,0.8,0.0)
```
Starts as dim ember when cold, scales to full flame as bed approaches target. `heaterfire` auto-disables when target is reached.

### 8.3 Print progress bar on dark background

```ini
layers:
    progress -1 0 add    (0.0, 0.5, 1.0)
    static    1  0 bottom (0.0, 0.0, 0.05)
```
Blue bar fills left-to-right tracking `virtual_sdcard.progress`. Dim blue-black background so the bar is visible against darkness. Negative rate on `progress` = fill behind the progress point.

### 8.4 Knight Rider / Cylon scanner (idle) [17]

```ini
layers:
    cylon 3 0 top (1.0, 0.0, 0.0)
```
Red light bounces back and forth. Classic, eye-catching.

### 8.5 Critical error alarm [22][44]

```ini
run_on_error: true
layers:
    strobe    1  1.5  add        (1.0, 1.0, 1.0)
    breathing 2  0    difference (0.95, 0.0, 0.0)
    static    1  0    top        (1.0, 0.0, 0.0)
```
Red base + red breathing + white strobe flashes. Unmissable. Fires automatically on MCU shutdown.

### 8.6 Homing flash with colour-per-axis

```ini
[led_effect homing_x]
autostart: true
endstops: x
leds:
    neopixel:leds
layers:
    homing 2 0 add (1.0, 0.0, 0.0)    # red flash for X

[led_effect homing_y]
autostart: true
endstops: y
leds:
    neopixel:leds
layers:
    homing 2 0 add (0.0, 1.0, 0.0)    # green flash for Y

[led_effect homing_z]
autostart: true
endstops: z, probe
leds:
    neopixel:leds
layers:
    homing 2 0 add (0.0, 0.0, 1.0)    # blue flash for Z/probe
```
Each axis flashes a different colour during homing — functional AND cool.

### 8.7 Mirrored inward comet (bed warming)

```ini
heater: heater_bed
leds:
    neopixel:leds (1-12)
    neopixel:leds (24-13)
layers:
    heater 1 60 add (0,0,1),(1,0.4,0)
    comet  1 5  top (0,0,1),(1,0.4,0),(1,0,0)
```
Comets sweep inward from both bed edges toward centre, with blue-to-orange heater-reactive colours.

### 8.8 Twinkle standby

```ini
layers:
    twinkle 80 15 top (0.2, 0.2, 0.4)
    static   0  0 bottom (0.01, 0.01, 0.02)
```
Gentle random sparkle on a near-black background. Very ambient.

---

## 10. Performance and gotchas

### 9.1 Frame rate [13][24][25]

- The plugin pre-renders most layer types at startup — only `fire` and `twinkle` render per frame
- Tested up to ~100 LEDs / 12 simultaneous layers on a Raspberry Pi 4 at 24 FPS [13]
- **"Neopixel update did not succeed"** error = MCU can't finish transmitting the LED frame before stepper interrupts fire. Mitigations:
  - Lower `frame_rate` (try 10-12 for bed LEDs during printing)
  - Offload NeoPixels to a secondary MCU (Pi Pico, Seeed XIAO, KlipperExpander) [24][25]
- For 24 LEDs on the Octopus Pro, 24 FPS should be fine — the data transmission is short

### 9.2 Octopus Pro PB0 pin note [31]

Some Octopus Pro board revisions have a silkscreen error where PB0 is electrically PB10. A PCB correction was made to fix DFU mode heater behaviour. If you ever get "PB0 not available" errors, try `PB10`. Current config uses `PB0` and it works — no change needed.

### 9.3 Effect stacking

Multiple effects can run simultaneously on the SAME LEDs. They're composited using blend modes, evaluated bottom-up. Use `REPLACE=1` when switching states to cleanly stop previous effects before starting new ones [20].

### 9.4 FADETIME best practices

- State transitions (idle → heating → printing): use `FADETIME=1.0` to `2.0` for smooth crossfades
- Urgent transitions (error): use `FADETIME=0` for instant activation
- Return to idle after print: use `FADETIME=3.0` to `5.0` for a gentle wind-down

### 9.5 `run_on_error` limitations [22][23]

- Only one effect with `run_on_error: true` should exist per LED chain (or they'll all fire and stack)
- Cannot be tested via macros — only fires on actual MCU shutdown
- The effect definition must be simple enough to run without klippy fully operational

---

## 11. Implementation roadmap

When ready to implement (after mechanical repairs), the changes needed are:

### Phase 1 — Fix the basics (no new effects)

1. Change `chain_count: 29` → `chain_count: 24` (match physical hardware)
2. Update all LED ranges from `(1-27)` / `(1-26)` → `(1-24)`
3. Remove logo and nozzle effects (LEDs 27-29 don't exist)
4. Remove `leds_logo` and `leds_nozzle` macros (dead code)
5. Delete commented-out dead code (~60 lines)
6. Rename file `led_marcos.cfg` → `led_macros.cfg` and update `[include]` in printer.cfg

### Phase 2 — Add new effects

1. Design effects for each printer state (see §7)
2. Add mirrored `(1-12)` + `(24-13)` ranges for symmetrical animations
3. Add `progress` layer for print tracking
4. Add `heaterfire` for bed warming
5. Add per-axis `homing` effects (automatic, no macro)
6. Add `run_on_error` critical error effect
7. Add `delayed_gcode` for idle/startup and return-from-print

### Phase 3 — Integration

1. Update PRINT_START to call LED state transitions at each phase
2. Update PRINT_END to trigger completion effect + delayed return to idle
3. Update PAUSE/RESUME/M600 for filament change indication
4. Test each state transition with a real print

### Phase 4 — Toolhead LEDs (future, if hardware is added)

1. Identify the correct EBB36 Gen2 RGB header pin
2. Add `[neopixel toolhead_leds]` section
3. Add toolhead-specific effects (nozzle illumination, logo)
4. Integrate toolhead effects into the state macros

---

## 12. References

1. [Klipper Configuration Reference — neopixel section](https://www.klipper3d.org/Config_Reference.html)
2. [Klipper Config Reference — neopixel search](https://www.klipper3d.org/Config_Reference.html?h=neopixel)
3. [Klipper PR #7000 — PCA9685 PWM controller support](https://github.com/Klipper3d/klipper/pull/7000)
4. [Klipper PR #5614 — PCA9685 host MCU support](https://github.com/Klipper3d/klipper/pull/5614)
5. [Klipper PR #3380 — raise neopixel chain limit](https://github.com/Klipper3d/klipper/pull/3380)
6. [Klipper G-Codes — SET_LED, SET_LED_TEMPLATE](https://www.klipper3d.org/G-Codes.html)
7. [Klipper Status Reference — print_stats, virtual_sdcard](https://www.klipper3d.org/Status_Reference.html)
8. [digitalninja-ro/klipper-neopixel — LED templates](https://github.com/digitalninja-ro/klipper-neopixel)
9. [klipper-neopixel/led_progress.cfg](https://github.com/digitalninja-ro/klipper-neopixel/blob/master/led_progress.cfg)
10. [julianschill/klipper-led_effect — main repository](https://github.com/julianschill/klipper-led_effect)
11. [klipper-led_effect/docs/LED_Effect.md — official docs](https://github.com/julianschill/klipper-led_effect/blob/master/docs/LED_Effect.md)
12. [julianschill/klipper-led_effect — DeepWiki overview](https://deepwiki.com/julianschill/klipper-led_effect)
13. [DeepWiki — LED effects pre-rendering and performance](https://deepwiki.com/julianschill/klipper-led_effect/4.1-led-effects)
14. [DeepWiki — configuration: leds section, index ranges](https://deepwiki.com/julianschill/klipper-led_effect/3-configuration)
15. [DeepWiki — effect configuration](https://deepwiki.com/julianschill/klipper-led_effect/3.2-effect-configuration)
16. [DeepWiki — example configurations](https://deepwiki.com/julianschill/klipper-led_effect/3.3-example-configurations)
17. [Voron Forum — "Knight Rider" LED effect in Klipper](https://forum.vorondesign.com/threads/klipper-led-effect-how-to-knight-rider-effect.460/)
18. [Siboor docs — configuring LED effects (blend modes)](https://docs.siboor.com/other-products/led-effect-notes/configuring-the-effects)
19. [bistory Gist — LED effects example configuration](https://gist.github.com/bistory/08e279ebda4d5cd2521dfe1add09dab4)
20. [DeepWiki — gcode commands](https://deepwiki.com/julianschill/klipper-led_effect/5.1-gcode-commands)
21. [DeepWiki — usage guide](https://deepwiki.com/julianschill/klipper-led_effect/5-usage-guide)
22. [Klipper Issue #2725 — led_effect run_on_error](https://github.com/KevinOConnor/klipper/issues/2725)
23. [klipper-led_effect Issue #64 — error config](https://github.com/julianschill/klipper-led_effect/issues/64)
24. [Klipper Discourse — issues with 45+ LED neopixel chain](https://klipper.discourse.group/t/issues-with-45-led-neopixel-chain/4026)
25. [klipper-led_effect Issue #56 — "neopixel update did not succeed"](https://github.com/julianschill/klipper-led_effect/issues/56)
26. [DeepWiki — installation](https://deepwiki.com/julianschill/klipper-led_effect/2-installation)
27. [klipper-led_effect install script](https://github.com/julianschill/klipper-led_effect/blob/master/install-led_effect.sh)
28. [Siboor docs — installing LED effects software](https://docs.siboor.com/other-products/led-effect-notes/installing-led-effects-software)
29. [BTT EBB36 GEN2 V1.0 — official wiki](https://global.bttwiki.com/EBB36_GEN2.html)
30. [BTT EBB36 CAN Toolhead Board — KB3D Wiki](https://wiki.kb-3d.com/en/home/btt/voron/BTT_EBB36)
31. [Voron Forum — Octopus Pro PB0 pin not working for neopixels](https://forum.vorondesign.com/threads/octopus-pro-pb0-pin-not-working-for-neopixels.754/)
32. [Voron Stealthburner official stealthburner_leds.cfg](https://github.com/VoronDesign/Voron-Stealthburner/blob/main/Firmware/stealthburner_leds.cfg)
33. [Voron docs — Stealthburner neopixel guide](https://docs.vorondesign.com/community/howto/drachenkatze/neopixel_guide.html)
34. [Voron Forum — klipper-led_effect main thread](https://forum.vorondesign.com/threads/klipper-led-effect.41/)
35. [klipper-led_effect examples — stealthburner BARF config](https://github.com/julianschill/klipper-led_effect/blob/master/examples/Voron_Stealthburner/stealthburner_led_effects_barf.cfg)
36. [klipper-led_effect examples — stealthburner BARF fan config](https://github.com/julianschill/klipper-led_effect/blob/master/examples/Voron_Stealthburner/stealthburner_led_effects_barf_fan.cfg)
37. [klipper-led_effect examples — stealthburner 3 LED config](https://github.com/julianschill/klipper-led_effect/blob/master/examples/Voron_Stealthburner/stealthburner_led_effects_3_leds.cfg)
38. [tanaes/whopping_Voron_mods — Rainbow Barf original config](https://github.com/tanaes/whopping_Voron_mods/blob/main/LEDs/Rainbow_Barf_Logo_LED/Code/stealthburner_led_effects_barf.cfg)
39. [Frix-x/klippain — Rainbow BARF issue #164](https://github.com/Frix-x/klippain/issues/164)
40. [RatOS documentation — LED control](https://os.ratrig.com/docs/configuration/led/)
41. [Siboor docs — configuring LED strips](https://docs.siboor.com/other-products/led-effect-notes/configuring-the-strips)
42. [Siboor docs — LED effect notes overview](https://docs.siboor.com/other-products/led-effect-notes)
43. [Laker87/klipper_config — leds.cfg on Octopus](https://github.com/Laker87/klipper_config/blob/octopus/leds.cfg)
44. [bistory Gist — LED effects example configuration](https://gist.github.com/bistory/08e279ebda4d5cd2521dfe1add09dab4)
45. [VzLight LED PCB — Vector 3D](https://vector3d.shop/products/vzlight-led-pcb)
46. [VzBoT docs — printer config](https://docs.vzbot.org/vz330_mellow/electronics/Printer_Config)
47. [jmlott/klipper-config — VzBot 330 AWD](https://github.com/jmlott/klipper-config)
48. [nofuturekid/klipper_config_vzbot_330](https://github.com/nofuturekid/klipper_config_vzbot_330)
49. [Braden5790/ZeroG_Mercury_One.1_Hydra](https://github.com/Braden5790/ZeroG_Mercury_One.1_Hydra)
50. [ethomasgt/Ender-5-Plus-Mercury-One-Klipper-Mainsail](https://github.com/ethomasgt/Ender-5-Plus-Mercury-One-Klipper-Mainsail)
51. [Printables — Klipper LED progress bar by Kotvic](https://www.printables.com/model/371310-klipper-led-progress-bar)
52. [Happy Hare Wiki — LED support](https://github.com/moggieuk/Happy-Hare/wiki/Led-Support)
53. [Happy Hare Wiki — LED support 2](https://github.com/moggieuk/Happy-Hare/wiki/Led-Support2)
54. [rootiest/zippy-klipper_config — comprehensive Voron config](https://github.com/rootiest/zippy-klipper_config)
55. [zippy stealthburner_led_effects_barf.cfg](https://github.com/rootiest/zippy-klipper_config/blob/master/machine/stealthburner_led_effects_barf.cfg)
56. [xkonni/klipper_config — neopixel.cfg Voron V2.4](https://github.com/xkonni/klipper_config/blob/voron-v2.4/neopixel.cfg)
57. [sttts/voron-klipper-config — stealthburner_leds.cfg](https://github.com/sttts/voron-klipper-config/blob/master/stealthburner_leds.cfg)
58. [gaehl/klipper_config — stealthburner.cfg](https://github.com/gaehl/klipper_config/blob/master/stealthburner.cfg)
59. [akinferno/VoronConfig — stealthburner_leds.cfg](https://github.com/akinferno/VoronConfig/blob/main/stealthburner_leds.cfg)
60. [bistory Gist — Voron Stealthburner LEDs](https://gist.github.com/bistory/1b1c7f39a19ed139971cdf6d38be6380)
61. [Gliptopolis/WLED_Klipper — WLED + Klipper guide](https://github.com/Gliptopolis/WLED_Klipper)
62. [Klipper Discourse — LED effects and macro help](https://klipper.discourse.group/t/led-effects-and-macro-help/18026)
63. [Klipper Discourse — neopixel LED as progress bar](https://klipper.discourse.group/t/neopixel-led-as-progress-bar/2710)
64. [Ellis' Print Tuning Guide — pause/resume/filament swaps](https://ellis3dp.com/Print-Tuning-Guide/articles/useful_macros/pause_resume_filament.html)
65. [Team FDM — Klipper filament unload / pause / M600 macros](https://www.teamfdm.com/forums/topic/1647-klipper-filament-unload-pause-m600-macros/)
66. [jschuh/klipper-macros — GCODE_ON_PRINT_STATUS](https://github.com/jschuh/klipper-macros)
67. [Voron Forum — klipper-led_effect page 3](https://forum.vorondesign.com/threads/klipper-led-effect.41/page-3)
68. [Moonraker DeepWiki — WLED integration](https://deepwiki.com/Arksine/moonraker/9.2-wled-integration)
69. [iamlite/WLED-Klipper-Helper](https://github.com/iamlite/WLED-Klipper-Helper)
70. [julianschill/moonshine — alternative LED control](https://github.com/julianschill/moonshine)
71. [Klipper Discourse — neopixel and SK6812](https://klipper.discourse.group/t/neopixel-in-klipper-and-sk6812/1256)
72. [Voron docs — stealthburner neopixel colour order guide](https://docs.vorondesign.com/community/howto/drachenkatze/neopixel_guide.html)
73. [klipper-led_effect Issue #237 — heater effect animation](https://github.com/julianschill/klipper-led_effect/issues/237)
74. [Klipper Discourse — gcode macros for LED colour changes during heating](https://klipper.discourse.group/t/gcode-macros-to-change-strip-led-colors-while-extruder-and-bed-reach-temperatures/761)
75. [klipper-led_effect Issue #100 — heater/stepper/progress simulation](https://github.com/julianschill/klipper-led_effect/issues/100)
76. [klipper-led_effect releases](https://github.com/julianschill/klipper-led_effect/releases)
77. [Team FDM — RGB headers and LED bars question](https://www.teamfdm.com/forums/topic/2319-question-about-rgb-headers-and-led-bars/)
78. [Gist by jfryman — EBB36 + U2C setup for Voron](https://gist.github.com/jfryman/0c3827079e23d7bc55f9677d2c6b8bec)
79. [Happy Hare Wiki — configuring mmu_hardware.cfg](https://github.com/moggieuk/Happy-Hare/wiki/Configuring-mmu_hardware.cfg)
80. [BIGTREETECH EBB 36/42 product page](https://biqu.equipment/products/bigtreetech-ebb-36-42-can-bus-for-bigtreetech-ebb-36-42-can-bus-u2c-v2-1-for-connecting-klipper-expansion-device-support-pt1000connecting-klipper-expansion-device)
81. [DeepWiki — ledEffect internal class reference](https://deepwiki.com/julianschill/klipper-led_effect/6.2-ledeffect)
82. [Voron Forum — LED effects separate thread](https://forum.vorondesign.com/threads/led-effects.821/)
83. [Klipper Discourse — neopixel configuration general discussion](https://klipper.discourse.group/t/neopixel-configuration/5928)
84. [RustedAperture/klipper-VzBot330 — generic VzBot config](https://github.com/RustedAperture/klipper-VzBot330)
85. [3DPTronics/BARF-Led-Kit — stealthburner_leds.cfg](https://github.com/3DPTronics/BARF-Led-Kit/blob/main/stealthburner_leds.cfg)
