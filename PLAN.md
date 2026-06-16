# Rev 3 design-review action plan

Issues identified during schematic assessment, in roughly descending order of "would-actually-matter." Status is tracked inline.

## 0. Only one 5V pin and one GND pin of J1 are connected

**Status:** ✅ done — wired in PCB via commit bf55525

**Issue.** The Pi 40-pin header J1 has two 5V pins (2, 4) and eight GND pins (6, 9, 14, 20, 25, 30, 34, 39). Only one 5V label and one Pi_GND label existed near J1's pin-side coordinates — the other 5V pin and seven GND pins were floating. This concentrated all input/return current through single contacts (~1A each rated), and forced switching-converter return current through a long high-inductance loop — bad for EMI and bad for the analog chain downstream.

**Action taken.** Added seven new `Pi_GND` labels (at J1 pins 9, 14, 20, 25, 30, 34, 39 — pin 6 was already connected) and one additional `+5V` label (at pin 2, to join the existing pin 4 connection), all placed directly at the J1 pin endpoints. Schematic was then synced into the PCB and the new pads routed with F.Cu point-to-point traces. PCB now has J1 pins 2 & 4 on `/+5V` and pins 6, 9, 14, 20, 25, 30, 34, 39 on `/Pi_GND`.

**Deferred sub-task — Pi_GND copper pour.** The original suggestion to lay a Pi_GND pour on the Pi-side region (instead of point-to-point routing) was *not* taken. The board still has zero zones and zero vias; all routing is F.Cu only. A pour would give a lower-impedance return path and an EMI win — worth revisiting if a future rev is spun. Keep-out across the U1 isolation line is mandatory if it's added.

## 1. R_BLEED1 sizing for NMH0515SC −15V rail

**Status:** ✅ resolved — bleeder changed from 2k2 → 1k5 for spec margin

**Issue.** The NMH0515SC datasheet (`KDC_NMH.F03` p.4, Application Notes) specifies minimum load = 10% of rated per rail. Rated current is 67 mA per rail, so minimum = 6.7 mA. R_BLEED1 at 2k2 drew 6.82 mA = 10.2% — right at the spec line with no tolerance margin. With ±5% resistor tolerance the worst case dipped to ~9.7% load, technically out of spec.

**Datasheet warning:** "*Lower than 10% minimum loading will result in an increase in output voltage, which may rise to typically double the specified output voltage if the output load falls to less than 5%.*" (i.e. −15V could become ~−30V transiently)

**Action taken.** Changed R_BLEED1 value from **2k2 → 1k5** in both the schematic Value property and the BoM. New load on −15V rail: 15V/1500Ω = 10.0 mA = ~15% of rated, with comfortable margin even at worst-case resistor tolerance. Dissipation 150 mW — fits the existing `R_Axial_DIN0207_L6.3mm_D2.5mm_P7.62mm_Horizontal` footprint (1/4–1/2W axial) without any PCB change.

**Blast radius.** Schematic Value + BoM only. PCB unaffected. Just stuff a 1k5 instead of a 2k2 when you assemble.

## 2. Output voltage at JST is not 9V under load

**Status:** ✅ resolved via documentation

**Issue.** R_9V1 (47Ω) is in series with the load. At ~20 mA total draw (two preamp channels) the drop is ~0.94V, so the JST outputs sit closer to ~8V than 9V. Labelling and naming say "+9V_filt_out" which is misleading.

**Action taken.** Added an "Output voltage under load" note to README's Design notes section. Did not change the circuit — the 47Ω is doing real work in the Stage 2 RC filter (~34 Hz corner with C_9V1) and the preamp is happy at 8V.

**Alternative not taken.** Could split into a smaller R (e.g. 10Ω) to land closer to 9V at the connector, but that raises the filter corner to ~160 Hz and lets more 7809 ripple through the audio band. Not worth it.

## 3. FB1 / FB2 footprint mismatch (BoM 0603 vs schematic 0805)

**Status:** ✅ resolved — BoM updated to match schematic + PCB

**Issue.** BoM listed 0603, but the schematic Footprint properties on FB1 (line 4798) and FB2 (line 5621) and both PCB footprints all said 0805. The PCB pads are 0805 and rev-3 boards are already fabricated, so 0805 is the authoritative reality.

**Action taken.** BoM CSV updated from `L_0603_1608Metric_Pad1.05x0.95mm_HandSolder` to `L_0805_2012Metric_Pad1.05x1.20mm_HandSolder` (line 7 of `rpi-preamp-psu-hat-BoM.csv`).

**Heads-up for ordering.** The schematic symbol description still mentions `BLM18AG601` as an example, but `BLM18` is Murata's 0603 series — it will NOT fit the 0805 pad on the PCB. When ordering, pick an 0805 (2.0×1.25 mm) ferrite bead with ~600 Ω @ 100 MHz; the Murata 0805 equivalent is `BLM21AG601` (or any equivalent from Würth, TDK, Bourns, etc.). The stale symbol-description text doesn't affect netlist or fab, just don't use it as a shopping reference.

## 4. 7809 input cap placement

**Status:** ✅ verified — no action needed, existing layout is correct

**Issue.** Wanted to confirm C_NMH1 (10 µF) and C_NMH2 (100 nF) are close enough to U2 pin 1 to act as effective input decoupling per the 7809 datasheet (≥0.33 µF at IN).

**Measured (from `rpi-preamp-psu-hat.kicad_pcb`):**
- U2 pin 1 (IN, +15V_f): (123.952, 74.041) mm
- C_NMH2 pin 1: (125.729, 66.168) mm — **trace length 8.6 mm** to U2 pin 1
- C_NMH1 pin 1: (122.555, 66.167) mm — **trace length 11.8 mm** to U2 pin 1

Both caps are well within the ~25 mm rule-of-thumb threshold. C_NMH2 (100 nF) handles HF stability at 8.6 mm; C_NMH1 (10 µF) provides bulk at 11.8 mm. The net routes on F.Cu only (no vias). Effective decoupling at U2 IN is excellent.

**Action.** None. The layout was already right — this item was a "verify by inspection" pass, and it passed.

---

# Rev 4 work

## R4-A. Replace OUT2 (JST) with barrel jack

**Status:** ✅ done — schematic in commit d4713eb, PCB in commit d1d199c

**Schematic** (`rpi-preamp-psu-hat.kicad_sch`):
- Added `Connector:Barrel_Jack` to the `lib_symbols` cache (copied verbatim from `Connector.kicad_sym`).
- Replaced OUT2 instance: `Connector:Conn_01x02_Pin` → `Connector:Barrel_Jack`, position (325.12, 52.07) → (313.69, 52.07), value "+9V_filt_out" → "Barrel_Jack_2.1mm_CN", footprint "Connector_JST:JST_XH_B2B-XH-A_1x02_P2.50mm_Vertical" → "Connector_BarrelJack:BarrelJack_CUI_PJ-102AH_Horizontal".
- Updated wires so the +9V_filt distribution lands on the new pin 1 location (321.31, 49.53). The existing ISO_GND label at (321.31, 54.61) already sits at the new pin 2 location.
- **Center-negative wiring:** Pin 1 (sleeve) = `+9V_filt_out`, Pin 2 (tip) = `ISO_GND` — guitar-pedal convention.

**PCB:**
- OUT2 footprint at (134.112, 91.59), 0° rotation. Jack opens toward the bottom board edge (+Y), with the opening face at Y ≈ 105.5 and the metal barrel protruding past the Edge.Cuts edge at Y = 106.27 as intended for edge mount.
- Pin 1 net = `Net-(OUT1-Pin_1)` (KiCad-auto-named — same node as OUT1 pin 1, i.e. the +9V_filt_out distribution); pin 2 net = `/ISO_GND`.
- OUT1 (JST, unchanged) at (124.5616, 92.456). Body-to-body clearance to OUT2 barrel ~2.45 mm — tight but workable.

**Known cosmetic issue:** the `+9V_filt_out` net label in the schematic isn't binding to the wire downstream of FB2/R_9V1, so the post-filter output node shows up in the PCB netlist as the auto-named `Net-(OUT1-Pin_1)` instead of `/+9V_filt_out`. Electrically correct. Worth fixing in the schematic for readability.

## R4-B. Add HAT mounting holes

**Status:** ✅ done — initial placement in commit d1d199c, X corrected to header centerline in a later commit

**Spec (per Raspberry Pi HAT board mechanical drawing, hat-board-mechanical.pdf):**
- Drill: 2.75 mm ±0.05 mm (M2.5)
- Land: ≥6.2 mm, isolated copper or bare board, **never connected to GND**
- Non-plated preferred
- **Geometry:** the GPIO header is centered on the 58 mm-wide mounting-hole rectangle, so the two top-row mounting holes sit on the header centerline (midway between the inboard and outboard pin rows) with **symmetric ±4.87 mm offsets** along the header's long axis from pin 1 and pin 39 respectively.

**Final placement:**
- Hole near J1 pin 1: **(151.13, 45.185)** — on header centerline X (between inboard X=149.86 and outboard X=152.4), 4.87 mm beyond pin 1 in the long axis (Y direction in our PCB)
- Hole near J1 pin 39: **(151.13, 103.185)** — same centerline X, 4.87 mm beyond pin 39
- Footprint: `MountingHole:MountingHole_2.7mm_M2.5_DIN965` (non-plated, `np_thru_hole`, no copper land — bare-board land naturally satisfies the "never connected to GND" spec).

**Placement history:**
- Initial (commit d1d199c): (153.36, 45.185) and (153.36, 103.185). Y was right, but X was on the outboard side of the outboard pin column instead of on the header centerline.
- Intermediate attempt: I shifted X to 151.13 (correct) but also moved Y to (48.685, 106.685) based on an over-literal reading of the HAT drawing's "29" dimension. That asymmetric placement pushed the top hole too close to J1 pin 1/pin 2 (1.87 mm center-to-center, 0.018 mm hole-to-hole clearance, triggering DRC errors) and forced the bottom hole past the board edge. Reverted.
- Final: keep the correct centerline X = 151.13, restore symmetric ±4.87 mm Y offsets.

**Board outline** (`Edge.Cuts` rect): (114.3, 42.0116) → (156.4894, 106.2736), ~42.2 × 64.3 mm. Unchanged from rev d1d199c. Still narrower than full HAT (65 mm wide) but mounting-hole-aligned with Pi.

## 5. No power-on LED, no input fuse, no reverse-polarity protection

**Status:** deferred to rev 4 (if at all)

**Issue.** None of these are required. They would be nice to have:
- Polyfuse on +5V input: protects the Pi rail if there's a short downstream.
- LED + current-limit resistor on the +9V output: "is it alive?" debug aid.
- Reverse-polarity protection: not really applicable — the HAT mates with the Pi via a fixed GPIO header, can't be reversed.

**Action.** None for rev 3. If a rev 4 spin happens, consider adding the polyfuse and LED.

## 6. C_NMH3 polarity on −15V rail

**Status:** ✅ resolved by inspection — polarity is correct

**Issue.** C_NMH3 is a polarized 4.7 µF on the −15V rail. Polarized caps on a negative-going rail need pin 1 (the "+" pin in KiCad's `Device:C_Polarized` symbol) tied to ISO_GND and pin 2 tied to −15V (nNEG15V) — so that the cap's positive terminal sees the higher potential (0V > −15V).

**Action.** Verified in schematic. (No action needed if already correct; flag for re-check during PCB silkscreen review.)

---

## Next steps

1. ⏸ Re-run DRC and ERC. The committed reports (`*-drc.rpt` 2026-06-12, `*-erc.rpt` 2026-06-15 15:20) predate today's PCB edits. Expect new checks against the barrel jack (silk/courtyard vs board edge at Y=105.4–106.27) and the mounting holes (copper/edge clearance).
2. ⏸ Schematic cleanup: re-bind the `+9V_filt_out` label so the post-FB2/R_9V1 wire stops auto-naming as `Net-(OUT1-Pin_1)` in the PCB.
3. ⏸ Item 5 (LED, polyfuse) still deferred — decide whether to fold into this rev 4 or push out.
4. ⏸ Optional: convert the J1 Pi_GND F.Cu fan-out into a Pi_GND pour for the EMI win that item 0 originally chased. Requires a keep-out across the U1 isolation line.
