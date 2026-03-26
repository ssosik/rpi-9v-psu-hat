# Extra tall hat stacking pin header

Yes, this stacking approach is entirely viable. Use the extra-tall stacking header (23mm body, 10mm+ pins) rather than the standard height version to ensure clearance over your electrolytics, and add M2.5 standoffs at the corner mounting holes. No changes to your KiCad design are needed — just specify the stacking header variant at assembly time.
Tall - 10mm
https://www.adafruit.com/product/2223

Extra Tall - 23mm
https://www.adafruit.com/product/1979

# When submitting to your fab, add a note: "Mouse bite breakaway tabs at board centreline — 0.5mm NPTH drills. Board can be separated into two 50×19mm units."

Or use the DAC2 ADC Pro

# Double Check component footprints match components in BOM


# Inspect previous builds and perform these tests:

How to Test This Circuit
Stage 1 — Before Powering On (Visual + Continuity)
Before applying any power, spend time at this stage. Most faults are catchable here for free.
Visual inspection:

Check every solder joint under magnification for bridges and cold joints, especially around Q3/Q4 which are tightly spaced
Verify C5 and C6 polarity — their positive terminals should face the XLR output side (away from the transistors), not the supply side
Verify C1 (bipolar electrolytic) is installed — it looks identical to the other radial caps but must be the BP type
Check Q1/Q2/Q3 are all BC559C (PNP) and Q4 is BC549C (NPN) — the packages are identical, easy to mix up
Verify transistor orientation — flat face of the TO-92 must match the PCB silkscreen

Multimeter continuity:

Measure resistance from +9V to GND with no power applied. Should be several kΩ (the bias resistor network). If it's near zero you have a short — don't power it
Check that J2 pin 1 (Mic+) and J2 pin 2 (GND) are not shorted together


Stage 2 — First Power-On (ESP's Recommended Safe Method)
The ESP assembly page describes this exact procedure for the 9V version:
Use a current-limited supply:
Put a 1kΩ resistor in series with the positive supply lead before connecting the battery or 9V supply. This limits inrush current and protects the transistors if there's a wiring error.
With the 1kΩ series resistor in circuit:

Connect power and measure voltage across the 1kΩ resistor
Expected drop: 3–4V, indicating a quiescent current draw of 3–4mA
If you see near-zero volts across the resistor, no current is flowing — check connections
If you see the full 9V across the resistor, something is shorted and drawing excessive current — disconnect immediately

DC bias check (the key measurement):
Once you've confirmed current draw is in range, measure DC voltage at the junction of Q3-C and Q4-C (the output node, between the transistors and C5/C6). With the 1kΩ series resistor still in circuit reducing supply to approximately 5V at the board:

Expected: ~3V DC at the collector junction

If you measure significantly more or less than 3V, the bias network (R3, R4, R5, R6, R7, R8, R9) has a fault. The exact value depends on transistor hFE spread — anywhere from 2–4V is acceptable.
Once satisfied, remove the 1kΩ series resistor and connect power directly.

Stage 3 — Functional Audio Test
Minimum kit needed:

9V supply
EM272 capsule (or any electret capsule) wired to J2
XLR cable to the HiFiBerry DAC+ ADC Pro, or just a multimeter on AC millivolts across J1 pins 2 and 3

Quick signal test with a multimeter:
Set your multimeter to AC millivolts. With the capsule connected and the board powered, tap the capsule gently or blow across it. You should see a few millivolts AC on the XLR output. This confirms the signal path is alive end-to-end.

With the HiFiBerry connected and BirdNET-Pi running:

Use arecord -l to confirm the card is recognised
Run arecord -D plughw:0,0 -f S16_LE -r 48000 -c 2 test.wav -d 5 to record 5 seconds
Play back with aplay test.wav and listen — you should hear ambient sound
Check the waveform in Audacity or similar to confirm it's not clipping or silent

Noise floor check:
With the capsule connected but in a quiet room, record a few seconds of silence. In Audacity, look at the waveform — you should see a flat line with only very fine noise. If you see 50/60Hz hum (a smooth sine wave), there's a ground loop or the 9V supply is noisy. If you see broadband hiss, that's normal and expected for an electret preamp.

Stage 4 — Simulation (Before or Instead of Building)
If you want to verify the schematic before committing to fab, KiCad's built-in SPICE simulator (via ngspice) can simulate this circuit. The transistors are well-supported — BC559 and BC549 SPICE models are widely available from Zetex/Diodes Inc.
You'd set up a transient analysis with a small AC source at the mic input and measure the output node. Expected voltage gain is approximately 1.6× (4dB) with R5=33kΩ as you have it.
Alternatively, LTspice (free from Analog Devices) is easier to use for this kind of discrete BJT circuit. A quick AC sweep will tell you the gain and bandwidth before you ever solder a component.

What to Do If It Doesn't Work
SymptomLikely causeNo current draw at allBad power connection, open circuit on supply railExcessive current (>10mA)Short circuit, wrong transistor polarityCollector voltage way off (near 0V or 9V)Wrong transistor type in Q1–Q4, C5/C6 reversed wrong wayLoud hum on outputNoisy 9V supply, ground loop — try a battery insteadVery low gain / weak signalR5 value too high, or capsule not powered correctlyOscillation / squealBypass cap missing or misplaced (C7), try adding small cap across Q1 emitter–collector as ESP suggests


# Cabling and connector considerations for remote mics+preamps assembly

The Three Connector Pieces You Need
1. On Your Preamp Board — Male XLR Panel Mount Connectors
Your J1 and J6 connectors on the preamp board are currently 2.54mm pin headers. For a permanent or semi-permanent installation you'd replace these with, or wire from them to, proper male XLR panel mount connectors — one for each channel.

The standard part is a Neutrik NC3MX or Switchcraft AAA3MZ — 3-pin male XLR, panel mount, solder cup termination. These are the industry standard and cost about $3–6 each.

If you want to keep the pin headers on the board and use a flying lead, you'd solder a male XLR plug (in-line, not panel mount) to a short pigtail — a Neutrik NC3MXX is the most common. These are the male connectors at the end of a cable.

For each channel the wiring at the XLR is:

XLR Pin	Signal	Your PCB
Pin 1	Shield/GND	J1/J6 pad 1 (GND)
Pin 2	Hot (+)	J1/J6 pad 2 (XLR2 / XLR4)
Pin 3	Cold (−)	J1/J6 pad 3 (XLR3 / XLR5)
2. The Cable — Standard Balanced Microphone Cable
A 2-conductor shielded twisted pair cable with XLR connectors on each end, sometimes called "mic cable" or "balanced XLR cable." Standard audio cable such as Mogami 2534, Canare L-4E6S (starquad), or any quality shielded twisted pair works perfectly. The internal construction for a standard 3-pin XLR cable is:

Pin 1 → Foil/braid shield (drain wire)
Pin 2 → Conductor 1 (usually white or red)
Pin 3 → Conductor 2 (usually black or blue)
For BirdNET-Pi field use, Canare starquad is a good choice because it has four conductors twisted together (pairs paralleled), which gives extra EMI rejection — useful if the cable runs near switching power supplies or WiFi hardware.

3. On the HiFiBerry — J4 Adapter Cable
J4 is a 6-pin connector that can be used to connect a balanced input such as XLR or 6.5mm jacks, with pin 1 on the left bottom corner. 
HiFiBerry

J4 is a 2×3 pin header at 2.54mm (0.1") pitch. You need to make a short adapter cable that breaks out from this header to your two XLR female connectors. The components are:

On the HiFiBerry end: A 2×3 2.54mm female crimp housing (Dupont/JST-style, or a Molex KK 396 2×3). You crimp individual female pins onto each wire and insert them into the housing, or buy a pre-crimped jumper harness and cut/re-pin as needed.

On the XLR end: Two female XLR panel mount connectors or female XLR in-line sockets — one for left, one for right. A Neutrik NC3FXX (in-line) or NC3FD (panel mount) for each channel.

The J4 wiring mapped to XLR female connectors:

J4 Pin	Signal	→	XLR Female	XLR Pin
1	Right −	→	Right XLR	Pin 3
2	Right +	→	Right XLR	Pin 2
3	GND	→	Right XLR	Pin 1
4	GND	→	Left XLR	Pin 1
5	Left +	→	Left XLR	Pin 2
6	Left −	→	Left XLR	Pin 3
Note that if you connect a source to the J4 pin header, you should not connect anything to the onboard 3.5mm phone jack — both inputs are connected in parallel. 
HiFiBerry

Complete Picture
[P93 Left Channel]                    [P93 Right Channel]
    J1 (male XLR pins/socket)             J6 (male XLR pins/socket)
         ↓                                      ↓
  Male XLR plug                          Male XLR plug
         ↓                                      ↓
  ~~~~ balanced XLR cable (any length up to ~50m) ~~~~
         ↓                                      ↓
  Female XLR socket                      Female XLR socket
         ↓                                      ↓
    [short adapter harness — ~10cm of wire]
         ↓
    [J4 on HiFiBerry DAC+ ADC Pro]
         ↓
    [Raspberry Pi 4B via GPIO header]
The simplest implementation for a BirdNET-Pi installation is to mount the Pi and HiFiBerry in a weatherproof box, bring two female XLR panel-mount sockets through the wall of the box, wire them to J4 internally via the short harness, and then run two standard XLR mic cables out to the preamp board wherever the microphones are sited.



# ✅ Duplicate and Mirror Preamp circuit to be one board board


# ✅ Add Stage 3 filtering on Preamp circuit

Remind me about "Stage 3 (local bypass at each preamp)"

12:51 PM
Stage 3 is the local bypass capacitor pair placed directly at each P93 preamp channel's V+ supply pin — physically on the preamp board, not the HAT.

What it is: One 10µF electrolytic and one 100nF ceramic, in parallel, at each preamp's power supply pin. With two channels (A and B), that's 4 capacitors total on the board.

Why it's needed even after all the HAT filtering: The 47Ω Stage 2 series resistor, which is responsible for all that attenuation, also has a downside — it creates impedance between the supply and the preamp. Any instantaneous current demand from the preamp (signal transients, gain changes) can't be met instantly through 47Ω, so without local bypass the supply voltage at the preamp pin will dip momentarily. The local caps provide that instantaneous charge locally, acting as a tiny reservoir right where it's needed, without the current having to travel back through the resistor.

The two-cap combination serves two different frequency ranges:

The 10µF electrolytic handles lower-frequency demands — charge reservoir for longer transients
The 100nF ceramic handles high-frequency transients, since electrolytics have poor HF impedance due to their internal inductance
Placement is critical: These caps must be physically located at the preamp's V+ pin on the PCB — within a few millimetres — not at the supply header or somewhere along the trace. A cap 20mm away from the pin it's supposed to bypass loses effectiveness at high frequencies because the trace inductance between them negates the benefit.

What it's not: It isn't redundant with the HAT filtering. The HAT stages remove noise from the supply; the local bypass ensures clean instantaneous current delivery. They solve different problems and both are needed.


