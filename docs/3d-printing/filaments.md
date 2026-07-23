---
icon: lucide/layers
title: Filaments
---

# Filaments

A reference for the most common FDM filaments — recommended settings, handling
notes, trade-offs, and what each one is actually good for.

!!! note "Settings are a starting point"

    The temperatures below are **typical ranges** compiled from manufacturer and
    community sources. Always defer to the number printed on the **spool/box** —
    every brand and colour is tuned slightly differently, and pigments (especially
    black, white, and glow) shift the ideal temperature. Run a
    [temperature tower](print-settings.md#temperature-tuning) when dialling in a
    new roll.

## Quick reference

| Filament | Nozzle (°C) | Bed (°C) | Enclosure | Part cooling | Difficulty | Hardened nozzle |
|---|---|---|---|---|---|---|
| **PLA** | 190–220 | 0–60 | No | 100 % | ★☆☆☆☆ | No |
| **PLA+** | 205–225 | 45–60 | No | 80–100 % | ★☆☆☆☆ | No |
| **Silk PLA** | 205–230 | 45–60 | No | 100 % | ★★☆☆☆ | No |
| **PETG** | 230–250 | 70–85 | Optional | 30–50 % | ★★☆☆☆ | No |
| **TPU (flex)** | 210–235 | 30–50 | No | 40–100 % | ★★★☆☆ | No |
| **ABS** | 230–250 | 90–110 | **Yes** | Off / low | ★★★★☆ | No |
| **ASA** | 235–260 | 90–110 | **Yes** | Off / low | ★★★★☆ | No |
| **Nylon (PA)** | 240–290 | 70–100 | Recommended | Off / low | ★★★★★ | If filled |
| **Polycarbonate (PC)** | 260–310 | 100–120 | **Yes** | Off | ★★★★★ | No |
| **PP (polypropylene)** | 220–250 | 85–100 | Recommended | Low | ★★★★★ | No |
| **PVB** | 200–220 | 70–90 | No | 100 % | ★★☆☆☆ | No |
| **CF / GF composites** | matrix +10 | per matrix | per matrix | per matrix | ★★★★☆ | **Yes** |
| **Wood / metal fill** | 190–220 | 45–60 | No | 100 % | ★★☆☆☆ | Metal fill: yes |
| **HIPS** (support/model) | 230–250 | 90–110 | **Yes** | Low | ★★★☆☆ | No |
| **PVA** (support) | 180–210 | 45–60 | No | 100 % | ★★★☆☆ | No |

★ = relative difficulty to print reliably (1 = easiest). "Matrix" = the base
polymer a composite is filled into (e.g. PA-CF is nylon + carbon fibre).

---

## PLA — Polylactic Acid

The default filament and the easiest to print. Made from plant starch, low
odour, minimal warping, prints without a heated bed.

| Setting | Value |
|---|---|
| Nozzle | 190–220 °C |
| Bed | 0–60 °C (adhesion improves at 55–60 °C) |
| Part cooling | 100 % |
| Print speed | 40–100+ mm/s (very forgiving) |
| Enclosure | Not needed — an enclosure can actually cause heat creep and clogs |

**Notes**

- Glass-transition temperature (~55–60 °C) is low: PLA **softens in a hot car,
  sunny window, or enclosure**. Not for anything load-bearing near heat.
- Brittle over time and under sustained stress; not ideal for living hinges or
  snap-fits that flex repeatedly.
- Only lightly hygroscopic — keeps well, but wet PLA gets brittle and prints rough.

<div class="grid" markdown>

!!! success "Pros"

    - Cheapest and most widely available
    - Excellent dimensional accuracy and fine detail
    - Low warping, no enclosure or heated bed strictly required
    - Low odour, plant-based

!!! failure "Cons"

    - Low heat resistance (softens ~55 °C)
    - Brittle; poor long-term UV and outdoor durability
    - Creeps/deforms under sustained load

</div>

**Use cases:** prototypes, miniatures, display models, toys, low-stress
brackets, anything printed indoors that won't get hot.

---

## PLA+ / Tough PLA

Vendor-specific PLA blends with added toughness modifiers. Prints almost like
standard PLA but is less brittle and has better layer adhesion. Runs slightly
hotter (205–225 °C). Same low heat resistance as PLA — the improvement is impact
strength, not temperature.

**Use cases:** functional-but-indoor parts, clips, brackets, RC parts, anything
where plain PLA cracks.

---

## Silk PLA

PLA with additives that give a glossy, satin sheen. Print a bit hotter
(205–230 °C) and **slower** for the best surface. The additives weaken layer
adhesion, so it's an **aesthetic** material, not a structural one. Often clogs
fine nozzles and can look stringy on multi-part surfaces.

**Use cases:** vases, decorative prints, cosplay props, anything where looks beat
strength.

---

## PETG — Polyethylene Terephthalate Glycol

The best all-round **functional** filament for most people: much tougher and more
heat/chemical resistant than PLA, but far easier than ABS. The glycol modification
of PET (the plastic in water bottles).

| Setting | Value |
|---|---|
| Nozzle | 230–250 °C |
| Bed | 70–85 °C |
| Part cooling | 30–50 % (too much cooling weakens layer bonding) |
| Print speed | 30–60 mm/s |
| Enclosure | Optional; helps on tall/thin parts |

**Notes**

- **Stringy.** Enable/dial in retraction and use a
  [temperature tower](print-settings.md#temperature-tuning) to reduce wisps.
- **Sticks *too well* to some beds** — PETG can rip chunks out of a bare PEI/glass
  bed. Use a thin glue-stick layer as a **release** agent, not just adhesion.
- Hygroscopic — absorbs moisture, which causes popping/bubbling and stringing.
  [Dry it](#storage-moisture) if a roll has been open a while.
- Good UV and water resistance; better outdoors than PLA (though not as good as ASA).

<div class="grid" markdown>

!!! success "Pros"

    - Strong, impact- and layer-adhesion-tough
    - Good temperature (~70–80 °C) and chemical/water resistance
    - Low warping, no enclosure required
    - Food-contact grades exist (single-use; layer lines still harbour bacteria)

!!! failure "Cons"

    - Stringing and blobbing need tuning
    - Can fuse to and damage the print surface
    - Absorbs moisture; benefits from dry storage
    - Slightly more flexible/less rigid than PLA

</div>

**Use cases:** functional parts, enclosures, brackets, outdoor fittings,
mechanical components, protective housings, printed tools.

---

## TPU / TPE — Flexible

Rubber-like elastic filament, sold by **shore hardness** (e.g. 95A is common and
printable; 85A is softer and much harder to feed). A **direct-drive** extruder is
strongly preferred — Bowden setups struggle to push flexible filament.

| Setting | Value |
|---|---|
| Nozzle | 210–235 °C |
| Bed | 30–50 °C |
| Part cooling | 40–100 % |
| Print speed | **15–30 mm/s** — slow and steady prevents jams |
| Retraction | Minimal (0–1 mm); flex filament buckles and coils in the extruder |

**Notes**

- Print slowly and with low/zero retraction. Rushing causes the filament to buckle
  and jam between the drive gear and hotend.
- Hygroscopic — dry it if it's been open.
- Softer shore ratings (down to 85A/70A) are dramatically harder to print.

<div class="grid" markdown>

!!! success "Pros"

    - Flexible, elastic, high impact and abrasion resistance
    - Great vibration damping and grip
    - Chemical- and oil-resistant grades available

!!! failure "Cons"

    - Slow to print; jam-prone on Bowden/soft grades
    - Stringing is common
    - Bridging and overhangs are weak

</div>

**Use cases:** phone cases, gaskets, seals, tyres/wheels, grips, vibration mounts,
wearables, cable strain reliefs.

---

## ABS — Acrylonitrile Butadiene Styrene

The classic engineering plastic (LEGO is ABS). Strong, heat-resistant, and
machinable, but **prone to warping and cracking** and it **emits styrene fumes** —
an enclosure and ventilation are effectively mandatory.

| Setting | Value |
|---|---|
| Nozzle | 230–250 °C |
| Bed | 90–110 °C |
| Part cooling | Off or very low (cooling causes layer splitting/warping) |
| Enclosure | **Required** — keeps ambient temperature high and even |

**Notes**

- Warps badly without a hot bed + enclosure. Large flat parts lift at the corners.
- Can be **vapour-smoothed with acetone** for a glossy, watertight finish.
- Prints best with a draft-free, warm chamber. Draughts cause cracking between layers.

!!! warning "Ventilation"

    ABS and ASA release styrene and ultra-fine particles (UFPs) while printing.
    Print in a **ventilated space or a filtered/vented enclosure**, not a bedroom or
    closed office. See [Safety](#safety).

<div class="grid" markdown>

!!! success "Pros"

    - Strong, tough, impact-resistant
    - Heat resistant (~100 °C)
    - Acetone-smoothable; machinable, glueable, paintable

!!! failure "Cons"

    - Warps and cracks; needs an enclosure + hot bed
    - Unpleasant/unhealthy fumes
    - Poor UV resistance (yellows/embrittles outdoors — use ASA instead)

</div>

**Use cases:** functional mechanical parts, automotive interior parts, enclosures,
tooling, parts exposed to moderate heat.

---

## ASA — Acrylonitrile Styrene Acrylate

Effectively **"ABS for outdoors."** Nearly identical print behaviour and strength,
but with **excellent UV and weather resistance** — it won't yellow or crack in
sunlight. Same enclosure and ventilation requirements as ABS.

| Setting | Value |
|---|---|
| Nozzle | 235–260 °C |
| Bed | 90–110 °C |
| Part cooling | Off or very low |
| Enclosure | **Required** |

**Use cases:** outdoor fixtures, automotive exterior parts, sensor/camera mounts,
planters, anything living in the sun. Prefer ASA over ABS whenever a part goes
outside.

---

## Nylon (PA) — Polyamide

Tough, semi-flexible, wear- and chemical-resistant engineering material with a low
coefficient of friction. **Extremely hygroscopic** — it drinks water from the air
and *must* be dried and often printed straight from a dry box, or it foams and prints
poorly.

| Setting | Value |
|---|---|
| Nozzle | 240–290 °C (higher for CF/GF-filled grades) |
| Bed | 70–100 °C |
| Part cooling | Off / low |
| Enclosure | Recommended (reduces warping, keeps it dry) |
| Adhesion | Garolite/PA-specific sheets or glue stick |

!!! warning "Nylon must be dry"

    Fresh-from-a-sealed-bag nylon still often needs drying. Wet nylon pops, oozes,
    strings, and loses most of its strength. Dry at ~70 °C for 6–12 h and print from
    an enclosure or dry box. See [Storage & moisture](#storage-moisture).

<div class="grid" markdown>

!!! success "Pros"

    - Very tough and durable; fatigue- and abrasion-resistant
    - Low friction — good for gears, bushings, living hinges
    - High heat and chemical resistance

!!! failure "Cons"

    - Soaks up moisture aggressively; needs active drying
    - Warps; benefits from an enclosure
    - Higher temps; can be tricky to get good bed adhesion

</div>

**Use cases:** gears, bearings, bushings, functional/mechanical parts, living
hinges, tooling, high-wear components.

---

## Polycarbonate (PC)

One of the **strongest and most heat-resistant** consumer filaments — high impact
strength and a heat-deflection point well above ABS. Also one of the **hardest to
print**: high temperatures, significant warping, and hygroscopic.

| Setting | Value |
|---|---|
| Nozzle | 260–310 °C (a hotend rated for high temp is required) |
| Bed | 100–120 °C |
| Part cooling | Off |
| Enclosure | **Required** (ideally actively heated) |

**Notes**

- Needs an all-metal hotend and often a hardened/high-temp setup.
- Warps strongly — enclosure, hot bed, and good adhesion are essential.
- Dry before printing; wet PC prints cloudy and weak.

**Use cases:** high-strength, heat-exposed, impact-resistant parts — machine
guards, electronics housings near heat, structural brackets, light lenses (optical
grades).

---

## Specialty & composite filaments

=== "Carbon-fibre / glass-fibre filled"

    Chopped carbon or glass fibre blended into a base polymer (PLA-CF, PETG-CF,
    PA-CF, PA-GF, etc.). The fibres add **stiffness and dimensional stability** and
    a matte finish, but make the filament **abrasive**.

    !!! danger "Abrasive — hardened nozzle required"

        CF/GF filament will grind a brass nozzle into an oversized, out-of-round
        hole within a spool or two. Use a **hardened steel, ruby, or tungsten
        nozzle** (0.4 mm minimum, 0.6 mm is friendlier).

    - Print settings follow the **base polymer**, usually +5–15 °C on the nozzle.
    - Fibres reduce warping and stringing versus the neat polymer.
    - Stiffer, **not** stronger in every axis — layer adhesion can be worse.

    **Use cases:** rigid brackets, drone/RC frames, jigs and fixtures, structural
    parts where stiffness and low weight matter.

=== "Wood / metal fill"

    PLA (usually) loaded with wood, cork, bamboo, or metal powder for aesthetics —
    real wood grain look/smell, or a brass/copper/steel sheen that can be polished.

    - Print like PLA (190–220 °C). Use a **larger nozzle (≥ 0.5 mm)** — the
      particles clog fine nozzles.
    - Metal-filled is **abrasive** → hardened nozzle.
    - Wood fill can be sanded and stained; darker "grain" appears at higher temps.

    **Use cases:** decorative pieces, props, busts, faux-wood/metal finishes.

=== "Glow-in-the-dark"

    Phosphorescent additives in a PLA/PETG base. Very **abrasive** (like a mild
    composite) — a hardened nozzle is recommended. Charge under light to glow.
    Prints slightly hotter than the base material.

    **Use cases:** signage, night-visible knobs/handles, decorative and novelty prints.

=== "PVB — Polyvinyl Butyral"

    A PLA-like material that can be **vapour-smoothed with isopropyl alcohol** (IPA)
    for a transparent, glossy finish — the "easy" way to get smooth prints without
    acetone. Prints around 200–220 °C. Slightly flexible and hygroscopic.

    **Use cases:** transparent/translucent parts, smoothed display pieces, light pipes.

=== "PP — Polypropylene"

    Lightweight, **fatigue- and chemical-resistant**, and semi-flexible — excellent
    living hinges that survive thousands of flexes. Notoriously **hard to get to
    stick** (packing tape or PP-specific sheets work because it bonds to itself).
    Warps; benefits from an enclosure.

    **Use cases:** living hinges, containers, chemical-resistant parts, automotive
    trim, fatigue-loaded snap-fits.

---

## Support materials

Dual-extruder / multi-material setups can print **dissolvable supports** in a
second material, leaving clean overhangs with no scarring.

| Material | Dissolves in | Pairs with | Notes |
|---|---|---|---|
| **PVA** | Water | PLA (and PETG) | Water-soluble; **extremely** hygroscopic — store bone-dry, print soon after opening. |
| **HIPS** | Limonene (d-limonene) | ABS | Also a fine standalone model material; needs an enclosure like ABS. |
| **BVOH** | Water | PLA, PETG, nylon | Dissolves faster and more reliably than PVA, but pricier. |

---

## Storage & moisture

Almost every filament is **hygroscopic** to some degree — it absorbs water from the
air. Wet filament causes popping/crackling sounds, steam, stringing, rough
surfaces, weak layer bonding, and clogs.

Ranked roughly worst → best at soaking up moisture:

> **Nylon / PVA ≫ PC > TPU > PETG > ABS/ASA > PLA**

!!! tip "Keeping filament dry"

    - **Store** rolls in sealed bins or vacuum bags with **silica gel** desiccant.
    - **Dry** wet filament in a filament dryer or oven: PLA ~45–55 °C, PETG ~60–65 °C,
      ABS/ASA/nylon/PC ~70–80 °C, for 4–12 h. Never exceed the material's
      glass-transition temperature or the roll will fuse.
    - Nylon, PVA, and PC often need to be **printed from a dry box** feeding the
      extruder, not just dried once.

## Safety

!!! warning "Fumes and particles"

    All FDM printing releases **ultra-fine particles (UFPs)** and volatile organic
    compounds (VOCs). Amounts are low for PLA/PETG but significant for **ABS, ASA,
    nylon, and PC** (styrene, etc.).

    - Print in a **ventilated room**, or use a **filtered/vented enclosure**,
      especially for ABS/ASA/nylon/PC. Don't sleep in the same room as an
      actively printing high-temp material.
    - Hot ends reach 200–310 °C and beds 100 °C+ — **burn risk**. Keep away from
      children and pets.
    - Don't treat "food-safe" filament grades as truly food-safe: layer lines trap
      bacteria and most nozzles/additives aren't certified. Use a food-safe sealer
      or a liner for anything that contacts food.

## Sources & further reading

- [Prusa Knowledge Base — Filament Material Guide](https://help.prusa3d.com/filament-material-guide)
- [Simplify3D — Ultimate 3D Printing Materials Guide](https://www.simplify3d.com/resources/materials-guide/)
- [Bambu Lab Wiki — Filament](https://wiki.bambulab.com/en/filament-acc/filament)
- [All3DP — 3D Printer Filament Types Guide](https://all3dp.com/1/3d-printer-filament-types-3d-printing-3d-filament/)
