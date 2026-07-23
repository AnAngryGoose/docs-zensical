---
icon: lucide/wrench
title: Troubleshooting
---

# Troubleshooting

Common FDM print defects, what causes them, and how to fix them. Symptoms often
overlap — work top-down (adhesion first) and change
[one setting at a time](print-settings.md).

## Quick diagnosis

| Symptom | Most likely cause | First thing to try |
|---|---|---|
| Print won't stick | Bed too far / dirty / cold | Re-level, clean with IPA, raise bed temp |
| Corners lifting | Warping (shrinkage + draughts) | Enclosure, brim, hotter bed, no fan (ABS) |
| Wispy hairs | Stringing (ooze) | Dry filament, tune retraction, lower temp |
| Gaps / thin walls | Under-extrusion | Check for clog, raise temp, calibrate flow |
| Blobs / rough surface | Over-extrusion | Lower flow, dry filament, tune temp |
| Layers split apart | Delamination | Hotter nozzle, less cooling, enclosure |
| Print knocked loose | Adhesion / collision | Brim, clean bed, check for warping |
| Ripples near edges | Ringing / ghosting | Slow down, tighten belts, input shaping |
| Bulge at the base | Elephant's foot | Lower bed temp, tune Z-offset, chamfer base |

---

## First layer won't stick

The most common failure. Almost always bed prep or Z height.

- **Re-level / set Z-offset.** Nozzle too high = lines don't adhere; too low = no
  plastic comes out. Use auto-level/mesh if available.
- **Clean the plate** with isopropyl alcohol — fingerprint grease is the #1 killer.
  Wash textured/PEI sheets with dish soap periodically.
- **Raise bed temperature** to the top of the material's range for the first layers.
- **Slow the first layer** (20–30 mm/s) and add a **brim**.
- Use the right surface (glue stick on glass/PEI for ABS/PC/nylon; glue as a
  *release* for PETG so it doesn't tear the sheet).

## Warping (corners curl up)

Plastic shrinks as it cools; uneven cooling pulls corners off the bed. Worst on
ABS/ASA/PC/nylon, occasional on big PETG/PLA parts.

- **Enclosure** to keep ambient temperature high and even (essential for ABS/ASA).
- **Hotter bed**, and turn the **part-cooling fan down/off** for ABS/ASA/nylon.
- **Brim or raft** for more grip; add **mouse-ears** to sharp corners.
- Eliminate **draughts** — no open windows or AC blowing across the printer.
- Clean bed + adhesive. Consider rounding/chamfering large flat corners in CAD.

## Stringing & oozing

Fine hairs strung between features as the nozzle travels.

- **Dry the filament** — moisture is the top cause, especially for
  [PETG, nylon, TPU, PVA](filaments.md#storage-moisture).
- **Tune retraction** (distance and speed) with a stringing test.
- **Lower the nozzle temperature** 5–10 °C.
- Enable **"combing"/avoid-crossing-perimeters** and reasonable travel speed.

## Under-extrusion

Too little plastic: gaps in walls, thin/weak layers, missing sections.

- **Clog or partial jam** — do a cold pull, check the nozzle, replace if worn.
- **Raise nozzle temp** so the plastic flows; slow the print if it can't keep up.
- **Calibrate flow / e-steps**; check the extruder isn't slipping or the spring
  tension is wrong.
- Check for a **worn or wrong-size nozzle**, or abrasive filament that oval-ed a
  brass nozzle (switch to [hardened](filaments.md#specialty-composite-filaments)).
- Make sure the filament path is clear and the spool turns freely.

## Over-extrusion

Too much plastic: blobs, rough/bulging top surfaces, dimensional bloat.

- **Lower flow rate / extrusion multiplier.**
- **Lower nozzle temperature** slightly.
- **Dry the filament** (wet filament over-extrudes and foams).
- Verify the correct **filament diameter** (1.75 mm) is set in the slicer.

## Layer separation / delamination

Layers visibly split or peel apart; part is weak along layer lines.

- **Raise nozzle temperature** for better layer bonding.
- **Reduce part cooling**, especially for ABS/ASA/PC/nylon — use an **enclosure**.
- **Dry the filament.**
- Lower layer height slightly, or reduce print speed so layers bond.

## Elephant's foot

The bottom few layers bulge outward wider than the model.

- **Lower the bed temperature** a few degrees.
- **Raise the Z-offset** very slightly (nozzle a hair higher on layer one).
- Enable **"elephant's-foot compensation"** in the slicer, or **chamfer** the
  bottom edge ~0.4 mm in CAD.

## Ringing / ghosting (echoes)

Repeating ripples/echoes on the surface after sharp features — vibration.

- **Slow down** print/travel speed and acceleration.
- **Tighten belts**; make sure the frame is rigid and on a stable surface.
- Enable **input shaping** (Klipper) or **linear/pressure advance**.

## Clogs & heat creep

Nozzle stops extruding mid-print or grinds.

- **Cold pull / atomic pull** to clear debris; poke with a cleaning needle.
- **Heat creep** — heat travelling too far up the heat break softens filament
  early. Check the **heatsink fan** is running; don't idle at temp for long; avoid
  enclosures for PLA.
- Dust/grit on the filament, or an all-metal hotend running too cool for the
  material, both cause jams.

## Pillowing / gaps in top surface

Holes or fuzzy, incomplete top layers.

- **Add top layers** (aim for ≈ 1 mm of solid, e.g. 5× 0.2 mm).
- **Increase infill density** so top layers have more to bridge onto.
- **Increase part cooling** so top bridges solidify.

## Spaghetti / print detached mid-way

A tangle of stringy plastic where the part came loose or a layer failed.

- Usually a **first-layer/adhesion** or **warping** failure that let the part pop
  off — fix adhesion (above).
- Tall thin models can topple: add a **brim**, supports, or print flat/orient better.
- Filament **runout or tangle** on the spool. Enable runout detection if available.

## When in doubt

1. Is the **filament dry**? Rule this out first — it mimics half the defects above.
2. Is the **first layer** clean and well-adhered?
3. Are the **temperatures** right for *this* spool ([temp tower](print-settings.md#temperature-tuning))?
4. Change **one** setting, reprint a small test, compare.

## Sources & further reading

- [Simplify3D — Print Quality Troubleshooting Guide](https://www.simplify3d.com/resources/print-quality-troubleshooting/)
- [Prusa Knowledge Base — Print Quality](https://help.prusa3d.com/category/print-quality-troubleshooting_225)
- [All3DP — Common 3D Printing Problems](https://all3dp.com/1/common-3d-printing-problems-troubleshooting-3d-printer-issues/)
