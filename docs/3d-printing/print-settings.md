---
icon: lucide/settings
title: Print Settings
---

# Print Settings

A practical reference for the slicer settings that matter most, what they trade
off, and sane starting values. Settings live in the slicer (OrcaSlicer,
PrusaSlicer, Bambu Studio, Cura); the names below are close across all of them.

!!! tip "Change one thing at a time"

    When tuning, adjust a **single** variable and reprint a small test. Changing
    five settings at once tells you nothing about which one helped.

## Sensible starting points

Good defaults for a **0.4 mm nozzle** in PLA/PETG. Tune from here.

| Setting | Draft / fast | Standard | Fine / detail |
|---|---|---|---|
| Layer height | 0.28–0.3 mm | 0.2 mm | 0.12 mm |
| Walls (perimeters) | 2 | 3 | 3–4 |
| Top/bottom layers | 3–4 | 4–5 | 5–6 |
| Infill | 10 % | 15–20 % | 15 % |
| Print speed | 150+ mm/s | 60–120 mm/s | 40–60 mm/s |

Rule of thumb: **layer height ≤ 75 % of nozzle diameter** (so ≤ 0.3 mm on a 0.4 mm
nozzle), and **line width ≈ 100–120 % of nozzle diameter**.

## Layers

- **Layer height** — the single biggest lever on quality vs. time. Halving it
  roughly doubles print time. 0.2 mm is the everyday default; drop to 0.12 mm for
  detail, raise to 0.28 mm for speed/strength on chunky parts.
- **First layer height** — often set thicker (e.g. 0.24–0.3 mm) for reliable
  adhesion, even when the rest is finer.
- **Line/extrusion width** — wider lines are stronger and faster; narrower resolve
  finer features. 0.42–0.45 mm is a good default on a 0.4 mm nozzle.

## Walls, top/bottom & infill

For most **functional** parts, **more walls beat more infill** — perimeters carry
load far more efficiently than internal lattice.

| Setting | Effect |
|---|---|
| **Walls / perimeters** | Outer shell loops. 2–3 for general use; 4+ for strong parts. The cheapest way to add real strength. |
| **Top/bottom layers** | Solid caps. Too few = pillowing/gaps on top surfaces. 4–5 layers (≈ 0.8–1 mm) is safe. |
| **Infill density** | 10–15 % for display parts, 20–30 % for functional, 40 %+ or solid only for high load. Diminishing returns above ~50 %. |
| **Infill pattern** | **Gyroid** — isotropic, strong in all directions, good default. **Grid/lines** — fast. **Honeycomb/cubic** — strong but slower. |

## Temperature tuning

Start from the [filament ranges](filaments.md) and refine with test prints.

- **Temperature tower** — a model with segments at descending temps (e.g. 230 → 200 °C
  in 5° steps). Pick the segment with the best surface, least stringing, and good
  overhangs/layer bonding. Most slicers can insert the temperature changes for you.
- Hotter = better layer adhesion and flow, but more stringing, oozing, and worse
  overhangs. Cooler = crisper detail but risk of under-extrusion and weak layers.
- **Flow rate / extrusion multiplier** — calibrate so walls are the intended
  thickness (a single-wall cube test). Over-extrusion causes blobs and rough tops;
  under-extrusion causes gaps.

## Speed

- Faster printing amplifies every other problem (ringing, poor adhesion,
  under-extrusion). The **outer wall** speed matters most for looks — keep it slower
  (e.g. 30–50% of infill speed) for a clean surface.
- Classic Marlin bed-slingers top out well below marketing numbers; **Klipper +
  input shaping** or CoreXY machines are what actually sustain 200+ mm/s.
- First layer is always printed **slow** (20–30 mm/s) for adhesion.

## Retraction & stringing

Retraction pulls filament back during travel moves so the nozzle doesn't ooze
wisps between features.

| Extruder | Retraction distance | Speed |
|---|---|---|
| **Direct-drive** | 0.5–1.5 mm | 25–45 mm/s |
| **Bowden** | 3–6 mm | 25–45 mm/s |

- Too much retraction → clicking, grinding, clogs, under-extrusion after travels.
- Print a **retraction/stringing test** to dial it in. Also lower nozzle temp and
  dry the filament — [wet filament](filaments.md#storage-moisture) is a top cause of
  stringing (especially PETG).

## Cooling (part-cooling fan)

| Material | Fan |
|---|---|
| PLA | 100 % (after the first 1–2 layers) |
| PETG | 30–50 % |
| ABS / ASA | Off or very low |
| Nylon / PC | Off / low |
| TPU | 40–100 % |

Cooling solidifies fresh plastic — essential for PLA overhangs and small layers,
but on ABS/ASA/nylon it causes layer splitting and warping. Reduce fan on very
small layers to avoid overheating a tiny area (or enable "minimum layer time").

## Supports

Overhangs steeper than roughly **45–60° from vertical** need support; shallower
usually print fine. Bridges over gaps often need none.

- **Support type** — *normal/grid* (everywhere) vs *tree/organic* (branches that
  use less material and mar the surface less).
- **Z distance / top gap** — the air gap between support and part. Larger = easier
  removal but rougher surface; ~0.1–0.2 mm is typical.
- **Support interface** — a dense top layer under the part for a cleaner supported
  surface.
- **Dissolvable supports** — PVA/BVOH (water) or HIPS (limonene) on a second
  extruder leave no scars. See [Filaments → Support materials](filaments.md#support-materials).
- Design to **avoid** supports where possible (chamfer overhangs, orient the part,
  split into pieces).

## Bed adhesion helpers

| Helper | Use it for |
|---|---|
| **Skirt** | Priming the nozzle and checking bed level before the part starts (doesn't touch the part). |
| **Brim** | Tall or small-footprint parts, or warp-prone corners. A flat collar around the base for extra grip. |
| **Raft** | A full sacrificial base under the part — good for rough beds or tricky materials; wastes plastic. |

Pair with the right **bed temperature**, a clean plate, and correct Z-offset — see
[Basics → Bed adhesion](basics.md#bed-adhesion-the-first-layer).

## Nice-to-have settings

- **Ironing** — a slow pass over top surfaces to smooth them (nice for flat tops,
  adds time).
- **Seam position** — where each layer starts/stops. "Aligned"/"back" hides the
  seam on one edge; "sharpest corner" tucks it into geometry.
- **Z-hop** — lifts the nozzle on travel to avoid knocking prints (can worsen
  stringing).
- **Adaptive/variable layer height** — thin layers on curved areas, thick on
  vertical walls, to balance detail and speed.

## Calibration order

When setting up a new printer or filament, calibrate roughly in this order:

1. Bed level / Z-offset → good first layer
2. Flow rate (extrusion multiplier)
3. Temperature tower
4. Retraction / stringing
5. Pressure advance / linear advance (Klipper/Marlin)
6. Speed and acceleration (input shaping on Klipper)

## Sources & further reading

- [Teaching Tech 3D Printer Calibration](https://teachingtechyt.github.io/calibration.html)
- [OrcaSlicer Calibration Wiki](https://github.com/SoftFever/OrcaSlicer/wiki/Calibration)
- [Prusa Knowledge Base](https://help.prusa3d.com/)
