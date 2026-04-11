# OrcaSlicer Printer Settings — Notes tab

> One file per tab. See [`README.md`](README.md) for the index of all tab documents and shared hardware context.
>
> The Notes tab is a **free-form text field** attached to the printer profile. It has no effect on slicing or gcode generation — it's purely for the user's own documentation. OrcaSlicer stores whatever you type as the `notes` key in the profile JSON, and displays it as a multi-line text area in the printer settings UI.
>
> **Our profile currently has this tab empty.** This document covers what to put there, not what to change.

---

## Current value

Nothing. Not set in `ZeroG Mercury One.json`, not set in the inheritance chain, so OrcaSlicer shows an empty text area.

---

## What the Notes tab is for

From the OrcaSlicer wiki and community usage, the Notes tab is commonly used for:

1. **Hardware provenance** — a record of the physical printer this profile matches (frame, toolhead, hotend, extruder, motherboard revision, etc.)
2. **Change log** — a short history of what's been changed in the profile and why
3. **Known quirks** — warnings to yourself about things that went wrong in the past or settings that look weird but are deliberate
4. **Cross-references** — pointers to external documentation (git repo, community profile source, tuning sheet)

None of this is functionally necessary — the slicer doesn't read it. But it **travels with the profile** when you export/share it. If you ever share your profile with another user (or with yourself on a fresh machine), the notes are the only context they have for why the settings look the way they do.

---

## Recommended content for this field

Given we now have a full documentation folder (`orca_slicer/printer/`), the most valuable thing to put in Notes is a **pointer to the documentation folder**, plus a brief summary of the critical findings. This way:

- Anyone opening the profile in OrcaSlicer sees a trail back to the full docs
- Quick-reference for the two most impactful findings (Motion Ability 4× mismatch, Extruder bad wipe combination)
- Doesn't duplicate the full docs, just points at them

### Suggested notes content

```
ZeroG Mercury One.1 + Nebula Pro frame
VzBot CNC toolhead (aluminium variant)
Phaetus Rapido 2 HF hotend
Orbiter 2.5 extruder (7.5:1, ~0.06mm backlash)
Beacon RevH probe
0.4mm hardened steel nozzle
Build volume: 260 x 245 x 258 mm

Profile documentation:
  https://github.com/alewis/3d-printer  (path: orca_slicer/printer/)

Critical reminders (see docs for full reasoning):
  - machine_max_acceleration_* should match Klipper max_accel (10000).
    Anything higher than that causes the slicer to plan gcode for
    headroom that doesn't exist. See motion_ability.md
  - wipe ON + retract_when_changing_layer ON. The combination of both
    OFF is Adam L's "do not" combination and causes seam gorges.
    See extruder.md
  - Klipper source of truth: ~/printer_data/config/printer.cfg on the Pi
    (192.168.x.x). Keep slicer motion limits in sync with printer.cfg
```

Substitute the github URL with the actual repo if/when it's pushed public.

---

## Recommendation

**Optional**: paste the suggested notes content above into the Notes tab in OrcaSlicer. It's not urgent — the documentation folder already has all this content — but it makes the profile self-documenting if it ever travels outside this repo.

**Not strongly recommended**: leaving it totally empty is also fine given the docs folder exists. The risk of leaving it empty is only if the profile gets copied to a machine without access to this repo.

---

## What Notes is NOT

A few things to clarify because they are sometimes confused with Notes:

| Thing | What it is | Where it lives |
|---|---|---|
| **Notes tab** | Free-form text on the printer profile. No functional effect | Printer settings → Notes (this tab) |
| **Filament notes** | Free-form text on a **filament** profile | Filament settings → Notes |
| **Process notes** | Free-form text on a **process** profile | Process settings → Notes |
| **Printer Notes field on Basic Info tab** | *(Older OrcaSlicer versions)* A short printer description shown in the printer picker. In newer versions this is part of the Basic Information tab | Basic Information tab |
| **Comments in machine_start_gcode / end_gcode / layer_change_gcode** | Text that gets written **into the output gcode file** as `; comments` | Machine G-code tab — see [`machine_gcode.md`](machine_gcode.md) |

If you want a note to appear in the output gcode (e.g. for post-processing or for humans reading the gcode), use comments in the Machine G-code tab, **not** this Notes tab.

---

## Summary of recommended Notes tab changes

| Action | Priority | Reason |
|---|---|---|
| Paste the suggested notes content above into the Notes field | Optional | Self-documents the profile if it ever travels outside this repo |
| Leave empty | Acceptable | The `orca_slicer/printer/` docs folder already covers everything |

**No critical or strongly-recommended changes.** This tab is effectively optional.

---

## References

[^orca-notes]: OrcaSlicer settings are stored as JSON profiles and the `notes` key is a free-form string that travels with the profile on export. The Notes tab UI displays it as a scrollable multi-line text area. No functional effect on slicing. Documented implicitly in the OrcaSlicer wiki under "Settings" → "Printer". <https://github.com/SoftFever/OrcaSlicer/wiki>
