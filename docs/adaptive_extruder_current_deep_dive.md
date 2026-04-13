# Adaptive Extruder TMC Current — Deep Dive Research

> Research document for dynamically adjusting the extruder stepper's `run_current` based on driver-area temperature, to prevent TMC2209 overtemperature warnings during prints in a heated chamber while retaining full-current torque during PLA prints at room temperature.
>
> **Status**: research only. Nothing implemented yet. Print running at time of writing — no printer changes made.

---

## Table of contents

1. [Summary](#1-summary)
2. [The problem](#2-the-problem)
3. [Critical finding — no pre-built community solution exists](#3-critical-finding--no-pre-built-community-solution-exists)
4. [Technical background](#4-technical-background)
5. [The five patterns the community uses](#5-the-five-patterns-the-community-uses)
6. [Proposed implementation — Pattern C with the EBB36_Driver thermistor](#6-proposed-implementation--pattern-c-with-the-ebb36_driver-thermistor)
7. [Draft implementation code](#7-draft-implementation-code)
8. [Hysteresis design](#8-hysteresis-design)
9. [Safety and gotchas](#9-safety-and-gotchas)
10. [Validation plan before committing](#10-validation-plan-before-committing)
11. [References](#11-references)

---

## 1. Summary

**The problem**: TMC2209 on the EBB36 toolhead hits `otpw=1(OvertempWarning!) t120=1` (die temperature > 120°C) during ASA+ prints at 60°C chamber ambient. Extruder `run_current: 0.850 A` is fine at room temp but excessive in a hot chamber.

**The research finding**: the exact pattern we want (extruder current that adapts to chamber/driver temperature during a print) is **not implemented publicly anywhere**. No Voron, VzBot, RatRig, Klippain, or Happy-Hare config does this. Klipper core has no native equivalent. It's a novel combination of well-established primitives.

**The closest existing primitives**:
- Klipper's `SET_TMC_CURRENT` (documented, safe, real-time)
- Klipper's `[delayed_gcode]` polling pattern (used for interruptible heat soak)
- Klipper's `printer["temperature_sensor X"].temperature` accessor
- Your existing `[temperature_sensor EBB36_Driver]` thermistor (already on the EBB36 PCB near the TMC2209)

**The proposed implementation**: a self-rescheduling `delayed_gcode` loop that polls `EBB36_Driver` temperature every 20 seconds and switches extruder current between three tiers (0.85 A / 0.65 A / 0.45 A) with hysteresis to prevent oscillation. Runs only when actively printing.

**Alternative if you don't want a polling loop**: Pattern B — pass `MATERIAL=ASA` from the slicer to `PRINT_START` and set current conditionally once at print start. Simpler, 80% of the benefit, doesn't adapt to chamber temp changes mid-print.

---

## 2. The problem

### Current symptom

During the 2026-04-12 ASA+ print (60°C chamber target), Klipper log filled with:

```
TMC 'extruder' reports DRV_STATUS: 001b0101 otpw=1(OvertempWarning!) t120=1 cs_actual=27
```

Repeated. This is the TMC2209 die temperature crossing ~120°C [1]. Not a shutdown yet (that's ~143-150°C), but a warning that it's heading there.

### Why it happens

- EBB36 Gen2 is physically inside the heated chamber
- Chamber at 60°C → ambient temperature for the TMC2209 is already elevated
- `run_current: 0.850 A` generates heat in the driver die proportional to I²
- At 60°C ambient + 0.85 A driver load, die temperature reaches 120°C

### Why not just lower run_current statically

Adam's priority: "I like that it's fast when printing PLA." The Orbiter 2.5 at 0.85 A delivers full torque for high volumetric flow PLA prints. Dropping to 0.60 A permanently works for hot chamber prints but may limit performance at room temperature.

### Why the driver not the motor

Important distinction: the `otpw` warning is about the **driver chip die temperature**, not the motor windings. The motor can be at 40°C and the driver die still at 120°C if the driver is working hard and has poor cooling. Our EBB36_Driver NTC thermistor is on the PCB near the driver chip, so it tracks driver heat — not motor heat.

### The physical constraint

Power dissipation in the driver scales with I²:
- 0.85 A → baseline heat (100%)
- 0.65 A → (0.65/0.85)² = 58% of baseline heat
- 0.45 A → (0.45/0.85)² = 28% of baseline heat

A two-tier switch (0.85 A in cool conditions, 0.45-0.50 A in hot chamber) halves to quarters the driver heat while still leaving enough current for Orbiter 2.5 to extrude reliably.

---

## 3. Critical finding — no pre-built community solution exists

After exhaustive research across Klipper Discourse, GitHub, Reddit, Voron / VzBot / RatRig / Klippain forums, and Klipper macro repositories, **no public configuration implements dynamic extruder current adjustment based on driver or chamber temperature during a print**.

### What DOES exist

- **`SET_TMC_CURRENT` in a homing macro** (official Klipper docs) [18] — save/reduce/restore pattern, exactly the primitives we need, but only used for pre/post-operation current change, not continuous adjustment
- **`SET_TMC_CURRENT` as two-level current in Happy-Hare** [36] — full-print current vs. MMU-operation current, but trigger is collision detection, not temperature
- **`temperature_fan` with `control: watermark`** [22] — Klipper-native temperature-driven fan control with hysteresis, but there's no equivalent `temperature_current` feature
- **`temperature_combined` sensor** [11] — combines multiple temperature sources for one reading, useful for multi-thermistor averaging but still drives fans only, not current
- **PR #6988 "Ready state current reduction"** [9] — OPEN pull request that would reduce current during idle, not merged, and addresses idle heat not in-print heat
- **PR #6769 "TMC2240 temperature report fix"** [8] — MERGED, makes TMC2240's internal temperature sensor usable via `temperature_combined`, but TMC2209 has no such sensor

### What does NOT exist

- **Klipper core**: no `temperature_current` or `temperature_stepper` module equivalent to `temperature_fan`
- **Kalico / Danger Klipper**: no equivalent feature
- **klipper_tmc_autotune**: has CoolStep (load-based current reduction), not temperature-based
- **jschuh/klipper-macros**: no SET_TMC_CURRENT at all
- **Klippain (Frix-x)**: requires static `run_current`, no adaptive support
- **garethky/klipper-voron2.4-config** (well-regarded Voron config with sophisticated heat soak): no SET_TMC_CURRENT anywhere
- **Voron official configs**: no adaptive current
- **VzBot community configs**: no adaptive current
- **RatOS**: no adaptive current
- **ZeroG / Mercury One community configs**: no adaptive current
- **Klipper Discourse search**: zero threads implementing this specific pattern

### Why it hasn't been done publicly

Likely reasons:
1. Most users solve it via static current reduction (simpler, fully stable, sacrifices performance)
2. Heated-chamber printing is a minority use case; most Klipper users print PLA at room-temp ambient where the issue doesn't arise
3. The `otpw` flag is a warning not an error — drivers survive extended operation at 120°C
4. The TMC2240 (with real temperature sensing) solves the same problem by having direct ADC access to die temp — solves a different way

### Implication for this work

Not reinventing the wheel — the wheel doesn't exist yet. What we'd build is novel in combination but built from thoroughly documented Klipper primitives used for other purposes. **Novelty ≠ risk**: each primitive (delayed_gcode polling, SET_TMC_CURRENT, temperature_sensor reading) is battle-tested in its own context.

---

## 4. Technical background

### TMC2209 temperature sensing — what's possible

TMC2209 exposes temperature only as **binary threshold flags** in DRV_STATUS [5][6]:

| Flag | Trigger | Meaning |
|---|---|---|
| `otpw` / `t120` | Die > ~120°C | Pre-warning (what we see) |
| `t143` | Die > ~143°C | Higher threshold |
| `t150` | Die > ~150°C | Shutdown approaching |
| `t157` | Die > ~157°C | Driver auto-disables |
| `ot` | — | Over-temperature shutdown active |

**There is NO continuous temperature reading from the TMC2209 die** [5]. Only threshold crossings. The only TMC driver that provides real-time die temperature is the **TMC2240** [6][8], which has on-chip ADC_TEMP. Upgrading to TMC2240 is an option but out of scope for this work.

### What we have instead

The EBB36 Gen2 PCB has a **physical NTC thermistor** positioned near the TMC2209, exposed in Klipper as:

```
[temperature_sensor EBB36_Driver]
sensor_type: Generic 3950
sensor_pin: EBB36:PA0
pullup_resistor: 2200
min_temp: 0
max_temp: 120
```

This reads the **PCB temperature near the driver chip**. It's not the same as die temperature (the die is hotter than the PCB trace), but it correlates. When the die hits 120°C and trips OTPW, the PCB thermistor typically reads 70-85°C depending on airflow and thermal coupling.

**Offset estimation**: community data [12] suggests die is typically 30-50°C hotter than the PCB thermistor at steady state under load. So OTPW at die=120°C → EBB36_Driver reading is typically ~75-85°C. For safety margin, target keeping EBB36_Driver **below 70°C** to ensure die stays below 110-115°C.

### Orbiter 2.5 motor specs

From OrbiterProjects.com [13][14][15]:

- Motor: LDO-36STH20-1004AHG (NEMA14 pancake, 1.0 A rated per phase)
- **Recommended Klipper `run_current: 0.850`** (RMS, = 1.2 A peak)
- **Recommended `hold_current: 0.100`** (but see Ref 10 — hold_current is no longer recommended)
- Sense resistor: 0.11 Ω
- Optimal motor temp: 65-75°C (not to exceed 85°C)
- Temperature rises linearly with ambient: +ΔT stays constant regardless of room temp
- "In case you print in an enclosed chamber, the stepper current needs to be reduced not to exceed 80-85°C stepper temperature"

Community-tested lower current values:
- **0.55 A minimum** — Voron forum, confirmed adequate for low-speed PLA [16]
- **0.5 A** — Voron official config for standard (non-Orbiter) extruders [29]
- **0.6 A** — Voron 2.4 community OTPW-fix value [25][26]

### SET_TMC_CURRENT safety

From official Klipper docs [1][4] and community confirmation [32]:

- Takes effect in real-time on TMC2209
- No 130ms standstill requirement (that's TMC5160/TMC2240 with StealthChop2 only [47])
- Can be called mid-print, mid-move without step loss concerns (for extruder specifically — XY/Z axis current changes are more risky [10])
- Reverts to printer.cfg defaults on MCU or driver reset
- Returns to current value after SAVE_CONFIG / restart

### `delayed_gcode` polling pattern

Klipper's canonical self-rescheduling pattern [2][20]:

```
[delayed_gcode my_loop]
initial_duration: 2
gcode:
    {% set temp = printer["temperature_sensor my_sensor"].temperature %}
    {action_respond_info("Temp: %.1f" % temp)}
    UPDATE_DELAYED_GCODE ID=my_loop DURATION=10
```

Starts 2 seconds after boot, runs every 10 seconds forever. Can be stopped with `UPDATE_DELAYED_GCODE ID=my_loop DURATION=0`. Most widely-used reference implementation is the "interruptible heat soak" pattern [20].

**Gotcha** [23]: inside a delayed_gcode block, do NOT use `TEMPERATURE_WAIT`, `M109`, or `M190` — these block the gcode queue and prevent the delayed_gcode from being rescheduled. Only use non-blocking reads (`printer.X.temperature`).

---

## 5. The five patterns the community uses

Ranked by adoption frequency observed during research:

### Pattern A — Static current reduction (MOST COMMON — >90% of responses)

Lower `run_current` in `printer.cfg` to a safe-for-worst-case value. Leave it there.

```
[tmc2209 extruder]
run_current: 0.600    # down from 0.850
```

**Pros**: zero complexity, fully stable, no macros.  
**Cons**: sacrifices PLA performance at room temp; 0.60 A may cause skipping at max volumetric flow on some filaments.  
**Adoption**: the "just fix it" solution used by nearly everyone.

### Pattern B — Material-conditional in PRINT_START (SECOND MOST COMMON)

Pass `MATERIAL=ASA` from slicer to PRINT_START macro, switch current once per print.

```gcode
[gcode_macro PRINT_START]
gcode:
    {% set MATERIAL = params.MATERIAL|default("PLA")|string %}
    {% if MATERIAL in ["ASA", "ABS", "PC", "NYLON"] %}
        SET_TMC_CURRENT STEPPER=extruder CURRENT=0.60
    {% else %}
        SET_TMC_CURRENT STEPPER=extruder CURRENT=0.85
    {% endif %}
    # ... rest of PRINT_START
```

Slicer sends: `PRINT_START BED_TEMP=100 EXTRUDER_TEMP=260 MATERIAL=ASA`

**Pros**: simple, switches once, clear intent, handles the common case.  
**Cons**: doesn't respond to chamber drift during long prints; doesn't handle mixed-material/multicolour prints with temp changes; requires slicer support for the `MATERIAL` parameter.  
**Adoption**: documented technique [42][43], rarely implemented specifically for current.

### Pattern C — Temperature-based polling via delayed_gcode (NOVEL — this proposal)

Self-rescheduling loop reads `EBB36_Driver` thermistor every ~20s, switches current based on thresholds with hysteresis.

**Pros**: truly adaptive, responds to chamber conditions, handles any material/situation, doesn't require slicer changes.  
**Cons**: bespoke implementation, slightly more complex, requires careful threshold + hysteresis tuning.  
**Adoption**: zero public implementations found. Novel in combination.

### Pattern D — OTPW flag polling (REACTIVE, not found implemented)

Poll `printer["tmc2209 extruder"].drv_status.otpw` in delayed_gcode. Only reduce current once OTPW has fired.

**Pros**: responds to actual driver state, not proxy temperature.  
**Cons**: by the time OTPW fires, the driver is already at 120°C — too reactive. Better as a secondary emergency reduction than primary mechanism.  
**Adoption**: theoretically discussed, zero implementations found.

### Pattern E — StealthChop vs SpreadCycle switching (mentioned, not adopted)

Switch TMC chopper mode based on temperature. Orbiter docs already recommend SpreadCycle (`stealthchop_threshold: 0`) for the extruder, so this optimization is already captured.

Not relevant to this printer (already on SpreadCycle for extruder per Orbiter spec [15]).

---

## 6. Proposed implementation — Pattern C with the EBB36_Driver thermistor

### The design in one paragraph

A `[delayed_gcode]` runs every 20 seconds. Each iteration reads the `EBB36_Driver` thermistor temperature. Compares against two thresholds (60°C and 75°C on the thermistor, corresponding to ~90°C and ~115°C at the die) with a 5°C hysteresis band to prevent oscillation. Sets extruder current to one of three levels (0.85 A / 0.65 A / 0.45 A). Tracks current tier in a `SET_GCODE_VARIABLE` to avoid redundant SET_TMC_CURRENT calls. Starts on boot (autostart via initial_duration) and runs continuously.

### Why EBB36_Driver instead of chamber_bottom

Two reasons:

1. **Directly measures what we care about**: the thermistor is physically next to the TMC2209 on the EBB36 PCB. It catches heat from any source (high ambient, high extrusion rate, poor airflow) — not just chamber ambient.

2. **Existing sensor, no hardware change**: already configured in `ebb_gen2.cfg` with the correct thermistor type and pullup.

Using `chamber_bottom` would miss cases where the driver heats up for non-ambient reasons (e.g., aggressive retraction during long PLA prints, stuck nozzle causing motor stall current, toolhead fan failure).

### Thresholds

Three tiers based on the research findings:

| EBB36_Driver reading | Estimated die temp | Current setting | Rationale |
|---|---|---|---|
| < 55°C (hysteresis: restore below 50°C) | ~75-85°C | **0.85 A** | Full Orbiter 2.5 spec current. Fast PLA territory |
| 55-70°C (transitions at 50/75) | ~85-110°C | **0.65 A** | Middle ground — still torquey enough for most flows |
| > 70°C (hysteresis: trip above 75°C) | ~100-120°C | **0.45 A** | Conservative — approaches OTPW region, 70% power reduction |

The die-temperature estimates are rough (community data [12] suggests +30-50°C offset between PCB thermistor and die). Tuning may be needed based on observed OTPW behaviour with the tiered current.

### Hysteresis band

**5°C gap between up-switch and down-switch thresholds** to prevent oscillation when temperature sits near a boundary:

- **Trip high → mid** at 55°C, restore mid → high below 50°C
- **Trip mid → low** at 75°C, restore low → mid below 70°C

So the "high → mid → high" oscillation requires the temperature to cross 55°C and then drop below 50°C — a 5°C swing. Mid → low → mid requires crossing 75°C and dropping below 70°C.

### Polling interval

**20 seconds**. Rationale:

- TMC2209 driver die thermal time constant is several seconds (the die has low thermal mass but PCB heat-sinking slows changes)
- 20s is faster than the die can heat/cool in meaningful increments
- 20s is slow enough to be negligible overhead (Klipper gcode queue has millisecond granularity; 20s intervals are trivial)
- Longer intervals (60s) risk missing rapid heat-ups from sudden high-flow moves

### Run condition

The loop should ONLY run during active printing. When idle, the extruder isn't drawing current at all (or minimal hold current), so the driver won't overheat. Polling during idle is wasteful.

Use `printer.idle_timeout.state` check [3]:

```
{% if printer.idle_timeout.state == "Printing" %}
    # ... adaptive logic ...
{% endif %}
# Always reschedule; the check inside handles whether to act
```

Or: gate activation via PRINT_START (start the loop) and PRINT_END (stop it). Simpler.

---

## 7. Draft implementation code

**NOT APPLIED.** This is the proposed code for review only.

```ini
# =============================================================================
# Adaptive extruder TMC current based on EBB36 driver-area temperature
# =============================================================================
# Addresses TMC2209 over-temperature warnings during hot-chamber prints
# (e.g. ASA+ at 60°C chamber).
#
# Design: delayed_gcode polling loop reads EBB36_Driver thermistor every 20s
# and switches extruder current between three tiers with 5°C hysteresis.
# Loop only runs during active printing (checked via idle_timeout.state).
#
# Thresholds (on EBB36_Driver thermistor readings):
#   <50°C: high current (0.85A)   — full Orbiter 2.5 spec
#   50-75°C: mid current (0.65A)  — still enough for most flows
#   >75°C: low current (0.45A)    — conservative, avoids OTPW
# Die temperature is typically 30-50°C hotter than the PCB thermistor reads.
#
# State tracking: tier variable avoids redundant SET_TMC_CURRENT calls.
# Hysteresis: up-trip at 55/75, down-trip at 50/70 — 5°C band.

[gcode_macro _adaptive_current_state]
variable_tier: "high"    # "high" | "mid" | "low"
gcode:
    # Sentinel macro for variable storage; body intentionally empty

[delayed_gcode adaptive_extruder_current]
initial_duration: 10
gcode:
    {% set state = printer.idle_timeout.state %}
    {% set driver_temp = printer["temperature_sensor EBB36_Driver"].temperature|float %}
    {% set current_tier = printer["gcode_macro _adaptive_current_state"].tier %}

    # Only adjust current during active printing
    {% if state == "Printing" %}
        {% if current_tier == "high" %}
            # From high: trip to mid at 55°C
            {% if driver_temp >= 55 %}
                SET_TMC_CURRENT STEPPER=extruder CURRENT=0.65
                SET_GCODE_VARIABLE MACRO=_adaptive_current_state VARIABLE=tier VALUE='"mid"'
                RESPOND MSG="Extruder current: 0.85 -> 0.65 A (driver temp {driver_temp|round(1)}°C)"
            {% endif %}
        {% elif current_tier == "mid" %}
            # From mid: trip to low at 75°C, restore to high below 50°C
            {% if driver_temp >= 75 %}
                SET_TMC_CURRENT STEPPER=extruder CURRENT=0.45
                SET_GCODE_VARIABLE MACRO=_adaptive_current_state VARIABLE=tier VALUE='"low"'
                RESPOND MSG="Extruder current: 0.65 -> 0.45 A (driver temp {driver_temp|round(1)}°C)"
            {% elif driver_temp < 50 %}
                SET_TMC_CURRENT STEPPER=extruder CURRENT=0.85
                SET_GCODE_VARIABLE MACRO=_adaptive_current_state VARIABLE=tier VALUE='"high"'
                RESPOND MSG="Extruder current: 0.65 -> 0.85 A (driver temp {driver_temp|round(1)}°C)"
            {% endif %}
        {% elif current_tier == "low" %}
            # From low: restore to mid below 70°C
            {% if driver_temp < 70 %}
                SET_TMC_CURRENT STEPPER=extruder CURRENT=0.65
                SET_GCODE_VARIABLE MACRO=_adaptive_current_state VARIABLE=tier VALUE='"mid"'
                RESPOND MSG="Extruder current: 0.45 -> 0.65 A (driver temp {driver_temp|round(1)}°C)"
            {% endif %}
        {% endif %}
    {% endif %}

    # Always reschedule, regardless of whether we're printing
    UPDATE_DELAYED_GCODE ID=adaptive_extruder_current DURATION=20
```

### What this does NOT do

- **No startup initialization**: assumes `run_current: 0.850` in `[tmc2209 extruder]` is the starting state, and `tier: "high"` in the state macro matches. After Klipper restart, the loop starts in "high" tier and will adjust down if needed.
- **No PRINT_END restore**: when print ends, the loop stops adjusting (idle_timeout.state != "Printing") but the tier variable retains its last value. Next print starts from whatever tier we left at. Acceptable but could be tightened.
- **No manual override**: if Adam wants to force a specific current mid-print, he can call `SET_TMC_CURRENT STEPPER=extruder CURRENT=X.XX` directly and the loop will immediately reset it based on temperature at the next iteration. Could add a "manual override" tier that bypasses the loop, but adds complexity.
- **No diagnostic output during normal operation**: only emits RESPOND messages on tier transitions. Won't clutter the console.

### Where it goes

In `printer.cfg` near the existing `[idle_timeout]` section, or in a new include file (e.g., `adaptive_current.cfg`) included from `printer.cfg`. Suggest the include-file approach for separation of concerns.

---

## 8. Hysteresis design

### Why hysteresis matters

Without a gap between up-trip and down-trip thresholds:

```
Temp at 55.0°C: switch high → mid (current 0.85 → 0.65)
→ 0.65 A generates less heat
→ Temp drops to 54.9°C within one iteration
→ Switch mid → high (current 0.65 → 0.85)
→ 0.85 A generates more heat
→ Temp rises to 55.0°C
→ Switch high → mid ...
```

Repeated switching every 20s is bad — thrashes the driver, fills the log with RESPOND messages, and each `SET_TMC_CURRENT` is a non-zero cost operation.

### The hysteresis band

5°C between up-trip and down-trip:

```
High tier (0.85 A):  held while temp < 55°C
Transition up:       temp ≥ 55°C  → drop to Mid
Mid tier (0.65 A):   held while 50°C ≤ temp < 75°C
Transition down:     temp < 50°C  → return to High
Transition up:       temp ≥ 75°C  → drop to Low
Low tier (0.45 A):   held while temp ≥ 70°C
Transition down:     temp < 70°C  → return to Mid
```

### Why 5°C specifically

- Typical room-to-chamber temperature gradient on this printer is tens of degrees
- EBB36_Driver thermistor noise is ~0.5°C, so 5°C is well above noise floor
- 5°C corresponds roughly to several minutes of thermal drift under load — plenty of margin
- Smaller gap (2°C) risks oscillation from noise + transient load spikes
- Larger gap (10°C) delays response to genuine temperature changes

### Alternative: time-based debouncing

Could also use minimum dwell time in each tier (e.g., "must remain in tier for 60s before switching"). Not proposed — hysteresis alone is sufficient and simpler.

---

## 9. Safety and gotchas

### 1. Mid-print current changes ARE safe for extruder

From Klipper docs [1] and community confirmation [32][46]:

- TMC2209 current changes take effect in real-time (no standstill calibration needed — that's TMC5160/TMC2240 only [47])
- Extruder is not held to a position like XY/Z axes are; under-extrusion is the only concern, not step loss
- At 0.45 A the Orbiter 2.5 still delivers enough torque for normal print speeds (community-confirmed [16])
- Changing current mid-move is supported

### 2. XY / Z current changes are NOT safe mid-print

Not a concern for this design (we only change extruder) but worth flagging: the "stepper_z1 and extruder otpw warnings" on this printer affect multiple drivers. If Adam ever wants to dynamically adjust Z driver current, it would need careful handling (momentary repositioning risk per Ref 10). Out of scope for this proposal.

### 3. `TEMPERATURE_WAIT` / `M109` / `M190` inside delayed_gcode blocks the loop

Already addressed in the design — we only use non-blocking reads (`printer.X.temperature`). No heater waits inside the loop [23].

### 4. State variable persists across prints

If a print ends with tier="low" and the chamber cools, the next print starts at 0.45 A until the loop upgrades it based on temp reading. This is usually fine (first 20s at low current just means slower priming), but to be cleaner could reset `tier` to "high" in PRINT_START.

### 5. State variable persists across Klipper restarts

`SET_GCODE_VARIABLE` values DO NOT survive Klipper restart. On startup, `tier` reverts to the declared default ("high"). This matches the `run_current: 0.850` default in `[tmc2209 extruder]`, so they stay in sync.

### 6. Manual `SET_TMC_CURRENT` gets overridden by the loop

If Adam calls `SET_TMC_CURRENT STEPPER=extruder CURRENT=1.0` manually during a print, the adaptive loop will reset it within 20s based on temperature. Not a bug, just behaviour to be aware of. If manual override is ever wanted, add a `SET_GCODE_VARIABLE VARIABLE=tier VALUE='"manual"'` mechanism to bypass the loop — but keeping it simple for v1.

### 7. PCB thermistor ≠ die temperature

The 30-50°C offset estimate from community data [12] is a range, not a precise value. Thresholds may need tuning based on observed behaviour:

- If OTPW still fires with the adaptive system in place, lower the 75°C threshold (e.g., to 65°C) — forcing earlier transition to low current
- If OTPW never fires and temps stay well below thresholds, raise them (e.g., 60/80) — more time in the high-current tier

First run should include log monitoring to verify thresholds are calibrated correctly.

### 8. What if the thermistor fails?

If `EBB36_Driver` reads an error or absurd value (e.g., > 120°C due to a disconnected sensor), Klipper will shut down at `max_temp` (already configured to 120). So the thermistor failing open would actually trigger Klipper's own protection before any damage. Failing closed (reading 0°C always) would keep the loop in "high" tier, returning to the original non-adaptive behaviour — not ideal but not dangerous.

### 9. Does it interfere with anything else?

- No conflict with `SHAPER_CALIBRATE` / `AXES_SHAPER_CALIBRATION` — those operate on XY, not extruder
- No conflict with `BED_MESH_CALIBRATE` — probes, not extruder
- No conflict with `SET_PRESSURE_ADVANCE` — different parameter
- No conflict with `PROBE_ACCURACY` — no extruder usage
- Does not interfere with the BTT safety relay, KAMP, or LED effects

### 10. Driver heat from IHOLD

During `SET_TMC_CURRENT`, only `run_current` (IRUN) is changed. The TMC2209 `hold_current` (IHOLD) is unchanged. Klipper defaults IHOLD to half of IRUN, so if run_current drops from 0.85 to 0.65, IHOLD drops proportionally from ~0.425 to ~0.325. This is generally correct behaviour — we WANT IHOLD to drop during hot conditions too. Worth verifying via `DUMP_TMC` after first deployment to confirm.

---

## 10. Validation plan before committing

If the decision is to proceed with this implementation, the rollout should be:

### Phase 1 — Test in dry-run mode (printer idle)

1. Deploy the config with `initial_duration: 10` but manually trigger a print simulation
2. Watch the console for RESPOND messages as the loop iterates
3. Verify the `_adaptive_current_state` variable updates correctly via `SET_GCODE_VARIABLE` visible in `printer["gcode_macro _adaptive_current_state"].tier`
4. Confirm no error messages, no unexpected side effects

### Phase 2 — Cool-state print

1. Start a PLA print at normal room temperature
2. Verify the loop stays in "high" tier throughout (`DUMP_TMC STEPPER=extruder` should show IRUN ≈ 0.85 A)
3. Verify no unexpected tier transitions

### Phase 3 — Hot-chamber print

1. Start an ASA+ print with 60°C chamber
2. Monitor console output for tier transitions
3. Verify OTPW warnings stop appearing (or appear less frequently) in klippy.log
4. Compare EBB36_Driver temperature trajectory against pre-implementation baseline
5. Verify no extrusion artefacts (under-extrusion, skipped steps)

### Phase 4 — Edge case validation

1. Simulate chamber temperature drop mid-print (e.g., open enclosure door briefly) to test hysteresis
2. Check that tier transitions happen smoothly without oscillation
3. Verify log doesn't fill with tier-change messages

### Phase 5 — Tune thresholds if needed

If Phase 3 still shows OTPW warnings:
- Lower the 75°C threshold to 65°C
- Lower the low-tier current from 0.45 to 0.40

If Phase 3 shows no OTPW and tier never transitions beyond "high":
- Threshold is too high; this print isn't hot enough to test
- Need a deliberately hot-chamber test

---

## 11. References

### Official Klipper documentation
1. [TMC_Drivers.md — SET_TMC_CURRENT, OTPW handling, motor sizing](https://www.klipper3d.org/TMC_Drivers.html)
2. [Command_Templates.md — delayed_gcode self-rescheduling pattern](https://www.klipper3d.org/Command_Templates.html)
3. [Status_Reference.md — accessible printer objects for macros](https://www.klipper3d.org/Status_Reference.html)
4. [G-Codes.md — SET_TMC_CURRENT syntax](https://www.klipper3d.org/G-Codes.html)

### TMC2209 hardware limitations
5. [Klipper Discourse — "How to get temperature TMC2209"](https://klipper.discourse.group/t/how-to-get-temperature-tmc2209/8767)
6. [Klipper Discourse — "How know driver temperature?"](https://klipper.discourse.group/t/how-know-driver-temperature/9233)

### TMC2240 and related developments
7. [Klipper Discourse — temperature_fan on TMC2240](https://klipper.discourse.group/t/temperature-fan-depending-on-the-tmc2240-temperature/15198)
8. [Klipper PR #6769 — TMC2240 temperature report fix (MERGED)](https://github.com/Klipper3d/klipper/pull/6769)
9. [Klipper PR #6988 — "Ready state current reduction" (OPEN)](https://github.com/Klipper3d/klipper/pull/6988)
10. [Klipper Discourse — hold_current deprecation discussion](https://klipper.discourse.group/t/stepper-motor-temperature-and-hold-current/24985)

### Temperature sensors and fan control
11. [Voron Forum — temperature_combined feature](https://forum.vorondesign.com/threads/new-temperature_combined-feature-in-klipper-to-cool-the-voron-0.928/)
12. [Klipper Discourse — CAN toolhead maximum temperatures](https://klipper.discourse.group/t/no-klipper-problem-maximum-can-bus-toolheads-temperatures-regarding-heated-chambers-question-poll/9940)

### Orbiter 2.5 motor specs
13. [OrbiterProjects.com — Orbiter v2.0 spec](https://www.orbiterprojects.com/orbiter-v2-0/)
14. [OrbiterProjects.com — "How hot is too hot?"](https://www.orbiterprojects.com/how-hot-is-too-hot/)
15. [Orbiter v2.0 firmware config PDF (trianglelab)](https://trianglelab.net/u_file/2112/11/file/Orbiterv20FirmwareConfiguration-031c.pdf)
16. [Voron Forum — LDO Orbiter v2 Klipper config](https://forum.vorondesign.com/threads/ldo-orbiter-v2-klipper-config-query.1648/)
17. [Ellis' Print Tuning Guide — motor currents](https://ellis3dp.com/Print-Tuning-Guide/articles/determining_motor_currents.html)

### Community SET_TMC_CURRENT patterns
18. [Klipper TMC_Drivers — sensorless homing SET_TMC_CURRENT pattern](https://www.klipper3d.org/TMC_Drivers.html) (canonical save/reduce/restore)
19. [EZ-Klipper-Macros — sensorless homing config](https://github.com/kyleisah/EZ-Klipper-Macros/blob/main/Config/Sensorless-Homing.cfg)
20. [Klipper Discourse — interruptible heat soak (delayed_gcode pattern)](https://klipper.discourse.group/t/interruptible-heat-soak/1552)
21. [Klipper Discourse — delayed_gcode loop implementation notes](https://klipper.discourse.group/t/trying-to-implement-a-loop-using-delayed-gcode/9579)
22. [Voron docs — chamber temp exhaust fan control](https://docs.vorondesign.com/community/howto/alchemyEngine/chamber_temperature_exhaust_fan.html)
23. [Klipper Discourse — delayed_gcode + TEMPERATURE_WAIT gotcha](https://klipper.discourse.group/t/delayed-gcode-temperature-wait/11829)

### Driver overheating discussions
24. [Klipper Discourse — TMC2130 OvertempError](https://klipper.discourse.group/t/tmc-2130-overtemperror/2040)
25. [Klipper Discourse — OTPW flag discussion (axis drivers)](https://klipper.discourse.group/t/stepper-driver-warning-otpw-flag-set/21681)
26. [Klipper Discourse — TMC stepper_y OTPW](https://klipper.discourse.group/t/tmc-stepper-y-drv-status-40130103-overtemperaturewarning-ve-overtemperatureerror/24278)
27. [Klipper Discourse — steppers overheating](https://klipper.discourse.group/t/steppers-overheating/22436)
28. [Klipper Discourse — stepper drivers getting too hot](https://klipper.discourse.group/t/steeper-drivers-getting-too-hot/22592)
29. Voron official configs — extruder run_current 0.5A standard (multiple repos)
30. [Klipper Discourse — EBB42 + NEMA14 heating problem](https://klipper.discourse.group/t/ebb42-can-and-nema14-stepper-heating-problem/16559)
31. [Rat Rig community — TMC2209 driver error](https://www.answeroverflow.com/m/1080160851610325082)

### TMC2209 current control behaviour
32. [Klipper Discourse — TMC2209 current (realtime confirmation)](https://klipper.discourse.group/t/tmc2209-current/9532)
33. [Klipper Discourse — DUMP_TMC interpretation](https://klipper.discourse.group/t/interpretation-dump-tmc-run-current/23860)

### Related features and near-misses
34. [klipper_tmc_autotune — CoolStep (load-based, not temperature-based)](https://github.com/andrewmcgr/klipper_tmc_autotune)
35. [Klipper Discourse — TMC Adaptive Microstep Table (unrelated)](https://klipper.discourse.group/t/tmc-adaptive-microstep-table/16652)
36. [Happy-Hare — collision-based two-level current (closest existing pattern)](https://github.com/moggieuk/Happy-Hare)
37. [garethky/klipper-voron2.4-config (no adaptive current)](https://github.com/garethky/klipper-voron2.4-config)
38. [jschuh/klipper-macros (no SET_TMC_CURRENT anywhere)](https://github.com/jschuh/klipper-macros)
39. [Frix-x/klippain (no adaptive current)](https://github.com/Frix-x/klippain)
40. [zellneralex/klipper_config (no adaptive current)](https://github.com/zellneralex/klipper_config/blob/master/macro.cfg)

### Slicer → PRINT_START patterns
41. [Klipper Discourse — filament thermal presets](https://klipper.discourse.group/t/assigning-a-filament-aka-thermal-preset-to-an-extruder/5849)
42. [Klipper Discourse — PRINT_START macro patterns](https://klipper.discourse.group/t/print-start-macro/6258)
43. [Voron docs — slicers and PRINT_START macros](https://docs.vorondesign.com/community/howto/EricZimmerman/SlicerAndPrintStart.html)

### Native Klipper temperature-driven features
44. [Klipper temperature_fan source](https://github.com/Klipper3d/klipper/blob/master/klippy/extras/temperature_fan.py)
45. [Klipper Issue #3949 — temperature_sensor as fan source](https://github.com/KevinOConnor/klipper/issues/3949)

### Mid-print current change safety
46. Klipper official TMC_Drivers.md — same as Ref 1, specific section on SET_TMC_CURRENT
47. Klipper docs — StealthChop2 130ms standstill (TMC5160/TMC2240 only)

### Hysteresis approaches
48. Klipper `temperature_fan` with `control: watermark` + `max_delta` — native hysteresis mechanism (same source as Ref 44)
49. [Klipper Discourse — interruptible heat soak state machine (same as Ref 20)](https://klipper.discourse.group/t/interruptible-heat-soak/1552)

### Community current values summary
50. Compiled from Refs 13, 14, 17, 25, 26, 27, 29, 31 — community-practice current values for TMC2209 extruders

---

*End of research document. Nothing has been applied to the printer. Implementation awaits user decision and testing protocol.*
