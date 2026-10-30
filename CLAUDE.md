# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

This is **not a software project** — it is a **KiCad 10.0 hardware design** for a Raspberry Pi HAT (`rpi-preamp-psu-hat`) that generates a clean, isolated +9V rail from the Pi's 5V GPIO for powering electret-mic preamps. See `README.md` for the design overview, stack-up, and component table — do not re-derive that content.

There are no build/lint/test commands. The "build" output is the gerber set in `gerbers/` (and `gerbers.zip`), regenerated from KiCad's Plot dialog when the PCB changes. ERC (schematic) and DRC (PCB) are run inside KiCad's GUI; there is no scripted CI.

## Design architecture (what future Claude needs to know)

The power chain is **five stages, in series**, and the schematic is laid out left-to-right in this order:

```
5V Pi ─► [Stage 1 RC filter] ─► [U1 NMH0515SC isolated DC-DC] ─► [LC filter] ─► [U2 MC7809CTG linear] ─► [Stage 2 RC filter] ─► JST OUT1/OUT2
        R_IN1 + FB1 + C_IN1/C_IN2        +15V / -15V / ISO_GND      L1 + C_NMH1/C_NMH2     +9V        R_9V1 + FB2 + C_9V1/C_9V2
```

Two facts about this topology drive most design decisions and should shape any suggestions:

1. **Galvanic isolation is the whole point.** The schematic has two distinct ground nets: `Pi_GND` (pre-U1) and `ISO_GND` (post-U1). They must never be bridged — not in the schematic, not on the PCB layout, not via mounting holes. The `+VIN` net is the pre-U1 isolated-side reference for U1's +Vin pin and is *not* the same as `+5V`.

2. **The −15V rail from U1 is unused** (only +15V feeds the 7809). `R_BLEED1` exists specifically to provide minimum load on the −15V rail so the NMH0515's cross-regulation works. Don't "clean up" the bleeder.

Net naming convention worth preserving: `+15V_f` = post-LC-filter +15V going into the 7809; `+9V` = raw 7809 output; `+9V_filt` = post-R_9V1 Stage 2 RC node; `+9V_filt_out` = post-FB2 output at OUT1 (JST) / OUT2 (barrel jack, center-positive). The `_f` / `_filt` suffixes carry meaning and should not be collapsed.

## File / directory conventions

- `rpi-preamp-psu-hat.kicad_{sch,pcb,pro,prl}` — the live design. Edit via KiCad only; the `.lck` files (`~*.lck`) appear while KiCad has the file open.
- `stereo-preamp-psu.pretty/` — project-local custom footprint library. `CP_Radial_D5.0mm_P5.00mm.kicad_mod` is referenced by `C_NMH1` (see schematic Footprint property `stereo-preamp-psu:CP_Radial_D5.0mm_P5.00mm`); `Logo_sosik_7.5mm.kicad_mod` is the front-silk logo (LOGO1 on the PCB), generated from `logo.kicad_sym`. The library is registered in `fp-lib-table` as `stereo-preamp-psu` via `${KIPRJMOD}`; do not delete it or those footprints will break.
- `rpi-preamp-psu-hat-backups/` — KiCad's auto-generated rolling backups (~400 zips). Ignore unless recovering from corruption.
- `gerbers/` and `gerbers.zip` — fab output. Regenerate together when the PCB changes so they stay in sync.
- `rpi-preamp-psu-hat-BoM.csv` — hand-maintained BoM. If you change a component value or footprint in the schematic, update this file too; it is what gets ordered against.

## Reference designators

This project uses **descriptive refdes** (e.g. `C_NMH1`, `C_REG2`, `R_BLEED1`, `R_9V1`) rather than sequential `C1, C2, R1`. Preserve this style when adding parts — the prefix names the stage/role (`NMH` = around U1, `REG` = around U2, `9V` = output stage, `IN` = input stage). Auto-annotation in KiCad will destroy this; don't run it without intent.

## License

CERN-OHL-S v2 (strongly reciprocal). Any derivatives need the same license and source release.
