# Raspberry Pi Preamp PSU HAT — Rev 4

A KiCad-designed Raspberry Pi HAT that generates a clean, isolated +9V supply from the Pi's 5V GPIO rail, intended to power low-noise electret microphone preamplifier circuits (such as the [ESP P93](https://sound-au.com/project93.htm)) for field audio recording applications like [BirdNET-Pi](https://github.com/mcguirepr89/BirdNET-Pi).

## Schematic

![Schematic](rpi-preamp-circuit-layout.png)

## PCB Layout

![PCB Layout](rpi-preamp-pcb-layout.png)

## Overview

The Pi's 5V supply rail is noisy — unsuitable for sensitive microphone preamp circuits without significant filtering and regulation. This HAT addresses that with a multi-stage power supply chain:

1. **Stage 1 — Input filter**: Ferrite bead + 4.7Ω series resistor + capacitors (100µF + 1µF) attenuate high-frequency noise on the 5V input and limit inrush from the Pi
2. **Isolated DC-DC conversion**: NMH0515SC converts 5V to ±15V with galvanic isolation, breaking any ground-loop path between the Pi and the preamp
3. **LC filter**: 100µH inductor + capacitors (10µF + 4.7µF + 100nF) reduce switching noise from the DC-DC converter
4. **Linear regulation**: MC7809CTG drops the rectified +15V to a clean, regulated +9V
5. **Stage 2 — Output filter**: 47Ω series resistor + capacitor bank (100µF + 100nF) + ferrite bead attenuates residual ripple before the output connectors

The filtered +9V is available on two paralleled outputs: **OUT1**, a JST XH 2-pin header, and **OUT2**, a 2.1mm DC barrel jack (**center-positive**, marked `CTR+` on the silkscreen).

## Key Components

| Designator | Part | Description |
|---|---|---|
| U1 | NMH0515SC | 5V → ±15V isolated DC-DC converter (Murata) |
| U2 | MC7809CTG | +9V linear regulator, TO-220 |
| L1 | 11R104C | 100µH radial inductor |
| FB1, FB2 | e.g. Murata BLM21AG601 | 0805 ferrite beads, ~600Ω @ 100MHz |
| J1 | 2×20 pin socket | Raspberry Pi GPIO header |
| OUT1 | JST XH B2B-XH-A | +9V filtered output, 2-pin |
| OUT2 | CUI PJ-102AH | +9V filtered output, 2.1mm barrel jack, center-positive |
| MH1, MH2 | M2.5 NPTH | Mounting holes on the Pi HAT hole pattern |

Full bill of materials: [`rpi-preamp-psu-hat-BoM.csv`](rpi-preamp-psu-hat-BoM.csv)

## Usage

### Hardware stack

```
[Raspberry Pi 4B]
      ↓ GPIO (5V + GND)
[This PSU HAT]
      ↓ JST XH or 2.1mm barrel (filtered +9V)
[P93 Stereo Preamp board]  ← electret capsules (EM272 or similar)
      ↓ balanced XLR
[HiFiBerry DAC+ ADC Pro]
      ↓ GPIO
[Raspberry Pi 4B]
```

### Stacking height

The HAT uses tall through-hole electrolytics (C_9V1, C_REG1: 100µF/16V, 8mm body). When stacking additional HATs, use an extra-tall stacking header to clear the tallest capacitors:

- **10mm stacking header** — [Adafruit #2223](https://www.adafruit.com/product/2223) (minimum clearance)
- **23mm stacking header** — [Adafruit #1979](https://www.adafruit.com/product/1979) (recommended)

Add M2.5 standoffs at the two mounting holes (MH1, MH2). They sit on the standard Pi HAT hole pattern beside the GPIO header.

## Design notes

A few non-obvious conventions to keep in mind before modifying the schematic or PCB:

### Grounding

There are **two distinct ground nets**: `Pi_GND` (everything upstream of U1) and `ISO_GND` (everything downstream of U1). Galvanic isolation across the NMH0515SC is the whole point of this board — bridging the two grounds anywhere (in the schematic, in PCB copper, or via mounting hardware) defeats the design. On the PCB, each domain has its own pour on both copper layers, with a 2.54mm copper-free channel between them. The mounting holes are non-plated with no copper land, so they are not connected to either ground. Keep it that way: a plated hole bonded to `Pi_GND` would let a metal standoff carry Pi ground into the case.

### Why R_BLEED1 exists

The NMH0515SC produces ±15V but only the +15V rail feeds the 7809 — the −15V rail is otherwise unloaded. `R_BLEED1` (1k5 across −15V → ISO_GND, ≈10mA ≈ 15% of rated load) provides minimum load on the unused rail so the converter's cross-regulation keeps the +15V rail stable. It is not optional; do not "clean it up."

### Output voltage under load

R_9V1 (47Ω) is in series with the load, so the output sits below 9V: at ~20mA (two preamp channels) the drop is ~0.94V, giving ~8V at OUT1/OUT2. This is intentional. The 47Ω is what sets the Stage 2 RC corner (~34Hz with C_9V1), and the P93 preamp is happy at 8V.

### Net naming

Suffixes on the rail names carry meaning:

| Net | Meaning |
|---|---|
| `+15V` | Raw output of U1 |
| `+15V_f` | Post-LC-filter, going into U2's input pin |
| `+9V` | Raw 7809 output |
| `+9V_filt` | After R_9V1 (Stage 2 RC node: C_9V1, C_9V2) |
| `+9V_filt_out` | After FB2, at the output connectors OUT1 / OUT2 |

When tracing the schematic, the `_f` / `_filt` suffix tells you which side of a filter stage a net is on.

### Reference designators

This project uses **descriptive refdes** (`C_NMH1`, `C_REG2`, `R_BLEED1`, `R_9V1`) rather than sequential `C1, C2, R1`. The prefix names the stage or role:

- `IN*` — Stage 1 input filter
- `NMH*` — around U1 (NMH0515SC)
- `REG*` — around U2 (the 7809 regulator)
- `9V*` — Stage 2 output filter
- `BLEED*` — the −15V bleeder

KiCad's auto-annotation will destroy these names — don't run it without intent.

### Custom footprint library

`stereo-preamp-psu.pretty/` is a project-local footprint library (registered in `fp-lib-table` via `${KIPRJMOD}`). It contains:

- `CP_Radial_D5.0mm_P5.00mm`: used by `C_NMH1`
- `Logo_sosik_7.5mm`: front-silkscreen logo, generated from `logo.kicad_sym`

Keep the directory alongside the project files. Deleting it breaks those footprint references.

## Files

| File | Description |
|---|---|
| `rpi-preamp-psu-hat.kicad_sch` | KiCad schematic |
| `rpi-preamp-psu-hat.kicad_pcb` | KiCad PCB layout |
| `rpi-preamp-psu-hat.kicad_pro` | KiCad project file |
| `rpi-preamp-psu-hat-BoM.csv` | Bill of materials |
| `rpi-preamp-psu-hat-erc.rpt` / `-drc.rpt` | Latest ERC / DRC reports |
| `logo.kicad_sym` | Logo artwork (bitmap2component output), source for the logo footprint |
| `PLAN.md` | Design-review log and open items |
| `gerbers/` | Gerber files ready for fab |
| `gerbers.zip` | Gerber zip for direct upload to PCB fab |

## Fabrication Notes

- Board is designed for standard 2-layer FR4, 1.6mm
- Through-hole parts on top; FB1/FB2 are the only SMD parts (0805, hand-solderable)
- Component reference/value silkscreen text is 0.6mm tall to fit the layout. That is below KiCad's default 0.8mm minimum, so DRC raises warnings. Confirm your fab's minimum before ordering

## License

Released under [CERN Open Hardware Licence v2 – Strongly Reciprocal (CERN-OHL-S)](https://ohwr.org/cern_ohl_s_v2.txt). Derivatives must be released under the same license with design files made available.
