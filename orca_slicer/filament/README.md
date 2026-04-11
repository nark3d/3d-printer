# OrcaSlicer filament profiles — documented

> One markdown file per physical filament spool we actually use. Each file lists every relevant filament-tab setting with: the OrcaSlicer default (walking the inheritance chain from `fdm_filament_common` → `fdm_filament_pla` → the bundled vendor profile), the researched community value (with footnote citations), our current value, and a comments field documenting the reasoning.
>
> Same format and research methodology as the [`process/`](../process/README.md) and [`printer/`](../printer/README.md) folders.

## Filament files

| Filament | File | Status |
|---|---|---|
| **SUNLU PLA Matte 1.75 mm** | [`sunlu_pla_matte.md`](sunlu_pla_matte.md) | ✅ Researched |

(More to come as we document other materials in use.)

## Shared context

- **Printer**: ZeroG Mercury One.1 + Nebula Pro frame (CoreXY, 260 × 245 × 258 mm)
- **Toolhead**: VzBot CNC toolhead (aluminium variant)
- **Hotend**: Phaetus Rapido 2 HF (claimed 45 mm³/s, practical PLA 20-24 mm³/s) [^rapido-realworld]
- **Extruder**: Orbiter 2.5 (7.5 : 1 geared, ~0.06 mm measured backlash)
- **Nozzle**: 0.4 mm hardened steel
- **Plate**: Textured PEI (default)
- **Material goal**: PLA, quality-first
- **Slicer version**: OrcaSlicer 2.0.x

## Inheritance chain

Each filament profile resolves through:

```
<our profile>.json                       (user overrides)
  → SUNLU PLA Matte @base                 (bundled OrcaFilamentLibrary, vendor-specific)
    → fdm_filament_pla                    (Orca PLA base defaults)
      → fdm_filament_common               (Orca all-filament base)
```

The "OrcaSlicer default" column in each filament file is the value Orca arrives at walking that chain *before* any of our overrides.

## Column meanings

Same as the process and printer docs:

- **Setting** — UI label as shown in OrcaSlicer
- **OrcaSlicer default** — value Orca arrives at without our overrides (including bundled vendor profile values if available)
- **Researched value** — community references / vendor docs / curated profiles, with footnote citations
- **Our value** — what our filament profile currently has set
- **Comments** — reasoning for our choice, especially when we differ from either the default or the researched value

## Per-filament vs per-process vs per-printer

Some settings exist on **all three** tabs (filament, process, printer). The resolution order is:

```
filament_<setting>     (highest priority — set per material)
  → <setting>           (process profile — per print mode)
    → printer <setting> (lowest priority — default for the printer)
```

Example: `filament_retraction_length: 0.5` (in the filament profile) will **override** `retraction_length: 1.0` (in the printer profile) **only when the filament profile value is set**. If the filament profile has `nil` for that field, the printer setting takes effect.

**This matters for audit** — the filament tab documents must look at what's overridden, what's inherited (marked `nil` in the JSON), and whether an override is intentional or cruft.

## References

[^rapido-realworld]: Phaetus Rapido 2 HF is specified for up to 45 mm³/s maximum volumetric flow. Real-world PLA printing typically achieves 20-24 mm³/s reliably — substantially lower than the manufacturer spec but still very high flow. <https://forum.vorondesign.com/threads/maximum-volumetric-flow-rate.195/>
