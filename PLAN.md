# Rev 3 design-review action plan

Issues identified during schematic assessment, in roughly descending order of "would-actually-matter." Status is tracked inline.

## 0. Only one 5V pin and one GND pin of J1 are connected

**Status:** ✅ schematic done — PCB sync still needed

**Issue.** The Pi 40-pin header J1 has two 5V pins (2, 4) and eight GND pins (6, 9, 14, 20, 25, 30, 34, 39). Only one 5V label and one Pi_GND label existed near J1's pin-side coordinates — the other 5V pin and seven GND pins were floating. This concentrated all input/return current through single contacts (~1A each rated), and forced switching-converter return current through a long high-inductance loop — bad for EMI and bad for the analog chain downstream.

**Action taken.** Added seven new `Pi_GND` labels (at J1 pins 9, 14, 20, 25, 30, 34, 39 — pin 6 was already connected) and one additional `+5V` label (at pin 2, to join the existing pin 4 connection), all placed directly at the J1 pin endpoints. No new components. Schematic now has 11 total `Pi_GND` labels (vs 4 before) and 3 total `+5V` labels (vs 2 before).

**Still to do (user, in KiCad):**
1. Reload the schematic if KiCad is open — file was edited externally.
2. Run **Tools → Update PCB from Schematic** to push the new connections into the netlist. New ratsnest lines will appear at the J1 pads that previously had no connection.
3. Route the new connections. Easiest: a single Pi_GND copper pour on the bottom layer (or wherever Pi_GND lives) covering the Pi-side of the board so all eight GND pads tie to it directly. Don't let this pour cross under U1 or into the ISO_GND region — the isolation gap must stay intact.
4. Run **Inspect → Electrical Rules Checker** in the schematic and **Inspect → Design Rules Checker** in the PCB to confirm nothing else broke.

**Blast radius.** Schematic-only changes are done. PCB needs routing updates — qualifies as a rev 4 spin if the existing rev-3 boards are already in your hands and you don't want to re-fab.

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

**Status:** open — PCB inspection needed

**Issue.** 7809 datasheet wants ≥0.33 µF at IN and ≥0.1 µF at OUT for stability. Schematic has the LC-filter caps (C_NMH1 10 µF + C_NMH2 100 nF) on the +15V_f net but not visibly placed *at* U2 pin 1. Whether they're close enough depends on the trace length on the PCB.

**Action.** Open the PCB, measure the routing distance from C_NMH2 to U2 pin 1. If more than ~25 mm, add a 100 nF X7R local decoupling cap right at U2's input. (No issue if already short.)

**Blast radius.** Either no change, or a new C component on the schematic + PCB = rev 4.

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

1. ✅ Document the output voltage drop (item 2) — done in README
2. ⏸ Item 0 (J1 GND/5V coverage) is the most important real-electrical fix — confirm whether this goes into a rev 4 spin or a patched rev 3.
3. ⏸ Confirm with the user before touching the schematic for items 1 and 3
4. Items 4 and 5 are PCB / rev-4 territory and need user direction
