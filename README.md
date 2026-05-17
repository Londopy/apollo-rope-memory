# 🧵 Core Rope Memory — Arduino Replica

A functional replica of 1960s NASA-era core rope memory, the read-only storage
technology that flew humans to the Moon aboard the Apollo Guidance Computer.
This project encodes binary data physically by weaving wires through ferrite
toroid cores and reads it back using electromagnetic induction, a hardware
comparator front-end, and an Arduino microcontroller.

> **“The software was woven into the hardware.”**
> — Margaret Hamilton, Lead Apollo Software Engineer

-----

## 📋 Table of Contents

- [Background](#-background)
- [How It Works](#-how-it-works)
- [Parts List](#-parts-list)
- [Circuit Overview](#-circuit-overview)
- [Step-by-Step Build Guide](#-step-by-step-build-guide)
- [Arduino Code](#-arduino-code)
- [Programming Your Memory](#-programming-your-memory)
- [Troubleshooting](#-troubleshooting)
- [Resume / Portfolio Notes](#-resume--portfolio-notes)
- [License](#-license)

-----

## 🚀 Background

Core rope memory was developed at MIT in the early 1960s for the Apollo
Guidance Computer (AGC). Unlike magnetic core RAM (which was read/write),
rope memory was purely ROM — software was physically woven into it by hand,
making it immune to power loss, radiation, and vibration.

The AGC held roughly 72 KB of rope memory, hand-woven by factory workers
at Raytheon under the direction of MIT engineers. The project lead,
Margaret Hamilton, earned the nickname **“Rope Mother”** for overseeing
this process.

This project replicates that technology at a small scale using modern
off-the-shelf components and an Arduino.

-----

## ⚙️ How It Works

Core rope memory encodes bits using the physical path of a wire:

```
Wire through the center of a core  →  Bit = 1
Wire routed around the outside     →  Bit = 0
```

When a current pulse is sent down the wire (the “word wire”), it induces a
voltage spike in any core it passes through — typically **200–900 mV** with
a 20-turn sense coil on a good ferrite. A hardware comparator detects this
spike and the Arduino captures it via a hardware interrupt.

Cores where the wire was routed around the outside see no induction — they
read as 0. This is entirely passive ROM: no power needed to retain data.

-----

## 🛠️ Parts List

|Part                |Spec / Part Number                       |Qty|Source|Est. Cost|
|--------------------|-----------------------------------------|---|------|---------|
|Arduino Uno         |Rev3                                     |1  |Amazon|$10.00   |
|Ferrite toroid cores|FT50-43 or FT50-77 (~12mm OD, type 43/77)|10 |Amazon|$10.00   |
|Enameled magnet wire|30 AWG, 1 oz spool (BNTECHGO or similar) |1  |Amazon|$8.00    |
|**Comparator**      |**LM393, DIP-8** *(not LM358 — see note)*|4  |Amazon|$4.00    |
|NPN transistor      |2N2222, TO-92 package                    |20 |Amazon|$3.00    |
|Signal diode        |1N4148, DO-35                            |20 |Amazon|$4.00    |
|Resistor assortment |100Ω, 1kΩ, 4.7kΩ, 10kΩ — 1/4W            |—  |Amazon|$5.00    |
|Ceramic capacitors  |100nF (104) disc type                    |10 |Amazon|$4.00    |
|Breadboard + jumpers|830 tie-point + M-M jumper kit           |1  |Amazon|$10.00   |

**Total: ~$58**

> ⚠️ **Do not substitute LM358 for LM393.** The LM358 is a general-purpose
> op-amp with a 0.3 V/µs slew rate — far too slow to detect sub-microsecond
> induction pulses. The LM393 is a dedicated comparator with ~300 ns response
> time and is the correct part. They are the same price and nearly pin-compatible.

> ⚠️ **Ferrite core material matters.** Use type 43 or type 77 MnZn/NiZn
> toroids (FT50-43 or FT50-77). Do **not** use iron-powder cores (T-xx series)
> or type 61 ferrite — their permeability is too low and signal will collapse
> by 10–16×. Clip-on ferrite chokes (the split kind for cables) also will
> not work.

### Optional but Highly Recommended

|Part                    |Notes                                       |Est. Cost|
|------------------------|--------------------------------------------|---------|
|DSO138 Oscilloscope Kit |Non-optional for debugging the sense circuit|~$15     |
|Helping hands / PCB vice|Holds cores while winding sense coils       |~$8      |
|Wire stripper / tweezers|For magnet wire work                        |~$5      |

-----

## ⚡ Circuit Overview

### Full Signal Chain

```
[ Arduino Digital Out Pin ]
          │
    [ 1kΩ resistor ]
          │
          ▼
  [ 2N2222 Transistor ]
  Collector ──→ [ Word Wire ] ──→ [ 100Ω ] ──→ +5V
  Emitter  ──→ GND
  (1N4148 flyback: anode→Emitter, cathode→+5V)

     [ Word Wire ]
     ├── Through Core 1  (Bit 7 = 1)
     ├── Around  Core 2  (Bit 6 = 0)
     ├── Through Core 3  (Bit 5 = 1)
     └── ... (through all 8 cores)


[ Sense Coil on Core N ]  (20 turns, 30 AWG, twisted-pair leads)
          │
    [ 100nF cap ]  ← AC-couple to block DC
          │
          ▼
  [ 2N3904 Preamp ]  ← common-emitter, gain ~30
  Base → sense signal (via 10kΩ)
  Collector → [ 4.7kΩ ] → +5V
  Emitter → GND
          │
          ▼
  [ LM393 Comparator ]
  (+) input: preamp collector
  (−) input: ~100mV reference (33kΩ / 1.5MΩ divider + 100nF bypass)
  Output: open-collector → [ 10kΩ pull-up ] → +5V
          │
          ▼
  [ Arduino Interrupt Pin (INT0 / INT1) ]
```

### Why Preamp + LM393 Instead of LM358

Two engineering problems make the LM358 fail here:

**Problem 1 — Slew rate.** The sense pulse is 200–500 ns long. The LM358
slews at 0.3 V/µs, which means it smears the pulse beyond recognition. The
LM393 responds in ~300 ns and snaps the output cleanly.

**Problem 2 — Sampling speed.** Arduino’s `digitalRead()` takes ~4 µs,
longer than the pulse itself. Using `attachInterrupt()` on INT0/INT1 lets
the hardware latch the edge immediately, regardless of what the CPU is doing.

### LM393 Reference Voltage Divider

```
5V ──[ 33kΩ ]──┬──[ 1.5MΩ ]── GND
               │
           (−) input of LM393  (~100 mV)
               │
           [ 100nF to GND ]  ← noise bypass
```

-----

## 🔧 Step-by-Step Build Guide

### Step 1 — Wind the Sense Coils

1. Cut 8 lengths of 30 AWG magnet wire, each about 40 cm long.
1. Wind **20 tight turns** through the center hole of each ferrite toroid.
   Use tweezers or a blunt needle to thread the wire.
1. Route the two leads as a **twisted pair** — twist them together for 5–10 cm
   before connecting to the circuit. This rejects common-mode noise from the
   nearby word wire.
1. Scrape the enamel off the last 5 mm of each lead using fine sandpaper,
   then tin the ends with solder.
1. Label each core 0–7 with a marker or small sticker.

> **Tip:** Keep turns tight and evenly spaced around the toroid.
> Loose, bunched-up windings reduce coupling and introduce stray capacitance.
> 
> **Don’t exceed ~25 turns** — past this, self-resonance distorts the pulse.

-----

### Step 2 — Weave the Word Wire

Decide what byte you want to store. For example, `10110001` in binary:

|Bit|Core|Value|Wire Routing  |
|---|----|-----|--------------|
|7  |1   |1    |Through center|
|6  |2   |0    |Around outside|
|5  |3   |1    |Through center|
|4  |4   |1    |Through center|
|3  |5   |0    |Around outside|
|2  |6   |0    |Around outside|
|1  |7   |0    |Around outside|
|0  |8   |1    |Through center|

Take a length of magnet wire and route it through or around each core in
sequence. This wire is your **word wire** — it represents one memory address.

To store multiple bytes, weave additional independent word wires through the
same 8 cores. Each wire is a separate address line driven by its own transistor.

> ⚠️ After weaving, keep the word wire physically separated from the sense
> coil leads. Capacitive coupling between the drive wire and sense lines is
> the #1 source of false “1” reads on unthreaded cores.

-----

### Step 3 — Build the Transistor Driver

For each word wire:

1. Connect the **Collector** of a 2N2222 to one end of the word wire.
1. Connect the other end of the word wire to **+5V via a 100Ω resistor**
   (limits drive current to ~30–50 mA for a strong sense pulse).
1. Connect the **Emitter** to GND.
1. Connect the **Base** to an Arduino digital output pin via a 1kΩ resistor.
1. Place a 1N4148 diode with **anode at Emitter, cathode at +5V** for flyback
   protection — do not skip this.

-----

### Step 4 — Build the Preamp Stage

For each of the 8 cores:

1. AC-couple the sense coil output through a **100nF capacitor**.
1. Connect the cap output to the **Base** of a 2N3904 via a 10kΩ resistor.
1. Connect the **Emitter** to GND.
1. Connect the **Collector** to +5V via a **4.7kΩ resistor**.
1. The Collector output feeds the LM393 (+) input.

-----

### Step 5 — Build the LM393 Comparator

For each of the 8 cores:

1. Connect the preamp Collector to the **(+) non-inverting input** of one
   LM393 channel.
1. Build the reference divider (33kΩ / 1.5MΩ from 5V to GND) and connect
   the midpoint to the **(−) inverting input**. Add 100nF from (−) to GND.
1. The LM393 output is **open-collector** — add a **10kΩ pull-up** to +5V.
1. Connect the output to an Arduino interrupt pin (D2 or D3) or digital input.
1. Add a **100nF decoupling cap** from each LM393 Vcc pin to GND.

The LM393 has 2 channels per chip — you need **4 chips** for 8 cores.

-----

### Step 6 — Wire Arduino Pins

```
Word Wire 0 (address 0x00) → Arduino D12 (via transistor)

Sense Core 0 (Bit 7)       → Arduino D2  ← INT0 (hardware interrupt)
Sense Core 1 (Bit 6)       → Arduino D3  ← INT1 (hardware interrupt)
Sense Core 2 (Bit 5)       → Arduino D4
Sense Core 3 (Bit 4)       → Arduino D5
Sense Core 4 (Bit 3)       → Arduino D6
Sense Core 5 (Bit 2)       → Arduino D7
Sense Core 6 (Bit 1)       → Arduino D8
Sense Core 7 (Bit 0)       → Arduino D9
```

> **First build tip:** Wire only Bit 7 and Bit 6 first (D2/D3 with interrupts)
> and validate those two bits completely before building out all eight. This
> cuts debugging time significantly.

-----

### Step 7 — Validate with Oscilloscope Before Coding

With everything wired, probe the sense coil output (before the preamp) while
manually triggering the word wire transistor:

- **You should see:** a spike of 200–900 mV, ~200–500 ns wide.
- **Nothing at all:** check coil continuity and word wire routing.
- **Spike under 50 mV:** confirm core is type 43 or 77, not iron-powder.
- **After preamp:** spike should be ~30× larger.
- **After LM393:** output should cleanly snap LOW when the spike arrives.

Do not skip this step. Debugging in software when the analog chain is broken
is where builds go to die.

-----

### Step 8 — Upload and Test

Upload the Arduino sketch below, open the Serial Monitor at 9600 baud, and
confirm the byte you wove matches what is printed.

-----

## 💻 Arduino Code

```cpp
// ============================================================
// Core Rope Memory Reader — Interrupt-Based
// 1-byte (8-bit) example
//
// D2 (INT0) and D3 (INT1) use hardware interrupts.
// D4–D9 use polled reads inside a gated sample window.
// LM393 output is active-LOW (open-collector + pull-up).
// ============================================================

// --- Pin Definitions ---
const int WORD_WIRE_PIN  = 12;
const int SENSE_PINS[8]  = {2, 3, 4, 5, 6, 7, 8, 9};

// --- Timing ---
const int PULSE_US       = 300;   // word wire pulse width (microseconds)
const int SAMPLE_US      = 50;    // read window after pulse falling edge

// --- Interrupt flags for D2 (Bit7) and D3 (Bit6) ---
volatile bool bit7_fired = false;
volatile bool bit6_fired = false;

void isr_bit7() { bit7_fired = true; }
void isr_bit6() { bit6_fired = true; }

// ============================================================
void setup() {
  Serial.begin(9600);

  pinMode(WORD_WIRE_PIN, OUTPUT);
  digitalWrite(WORD_WIRE_PIN, LOW);

  for (int i = 0; i < 8; i++) {
    pinMode(SENSE_PINS[i], INPUT);
  }

  // LM393 output goes LOW when a sense pulse is detected (active-low)
  attachInterrupt(digitalPinToInterrupt(2), isr_bit7, FALLING);
  attachInterrupt(digitalPinToInterrupt(3), isr_bit6, FALLING);

  Serial.println("================================================");
  Serial.println("  Core Rope Memory Reader — Ready");
  Serial.println("  INT0=Bit7 (D2)   INT1=Bit6 (D3)");
  Serial.println("================================================");
}

// ============================================================
byte readWordWire() {
  byte result = 0;

  // Clear interrupt flags before pulsing
  bit7_fired = false;
  bit6_fired = false;

  // 1. Fire the word wire pulse
  noInterrupts();
  digitalWrite(WORD_WIRE_PIN, HIGH);
  delayMicroseconds(PULSE_US);
  digitalWrite(WORD_WIRE_PIN, LOW);
  interrupts();

  // 2. Brief window for comparator outputs on D4–D9 to settle
  delayMicroseconds(SAMPLE_US);

  // 3. Collect interrupt-latched bits (D2, D3)
  if (bit7_fired) result |= (1 << 7);
  if (bit6_fired) result |= (1 << 6);

  // 4. Poll remaining sense pins (D4–D9)
  // LM393 is active-LOW: LOW = pulse detected = bit is 1
  for (int i = 2; i < 8; i++) {
    if (digitalRead(SENSE_PINS[i]) == LOW) {
      result |= (1 << (7 - i));
    }
  }

  return result;
}

// ============================================================
void printByte(byte val) {
  Serial.print("  Binary : 0b");
  for (int i = 7; i >= 0; i--) {
    Serial.print((val >> i) & 1);
    if (i == 4) Serial.print(" ");
  }
  Serial.print("\n  Hex    : 0x");
  if (val < 16) Serial.print("0");
  Serial.print(val, HEX);
  Serial.print("\n  Decimal: ");
  Serial.println(val, DEC);
}

// ============================================================
void loop() {
  byte data = readWordWire();
  Serial.println("------------------------------------------------");
  Serial.println("  Read result:");
  printByte(data);
  delay(2000);
}
```

-----

## 🧵 Programming Your Memory

To change the stored data, physically re-weave the word wire:

- **Store a 1** → thread the wire through the center hole of the core
- **Store a 0** → route the wire around the outside of the core

To store multiple bytes, wire an additional transistor driver per word wire
to a new Arduino output pin. Extend the sketch to pulse each address line
in sequence and print each result.

-----

## 🐛 Troubleshooting

|Symptom                          |Likely Cause                           |Fix                                                          |
|---------------------------------|---------------------------------------|-------------------------------------------------------------|
|All bits read as 0               |No sense signal reaching Arduino       |Probe sense coil with scope — check coil continuity first    |
|All bits read as 1               |Word wire coupling into all sense lines|Route word wire away from sense leads; use twisted sense pair|
|Weak/no spike on scope (<50 mV)  |Wrong ferrite material                 |Confirm cores are type 43 or 77 — not iron-powder or type 61 |
|Spike present but comparator flat|LM393 threshold too high, or bad wiring|Lower reference divider; verify LM393 pinout and pull-up     |
|Intermittent reads on D4–D9      |Polled read missing short pulse        |Increase PULSE_US; or add 74HC74 latch on comparator output  |
|Correct pattern, wrong bit order |Bit index inverted in firmware         |Swap `(1 << (7-i))` to `(1 << i)` in the polling loop        |
|Random false 1s                  |Breadboard noise / capacitive coupling |Add 100nF decoupling on LM393 Vcc; twist all sense leads     |
|Transistor not switching         |Wrong 2N2222 pinout                    |Pinout varies by brand — verify EBC order on your datasheet  |


> **Scope rule:** Clean 200+ mV spike at sense coil but Arduino reads nothing
> → problem is in the comparator circuit (threshold, pull-up, pinout).
> No spike at all → problem is mechanical (coil winding or word wire routing).

-----

## 📄 Resume / Portfolio Notes

This project can be listed on a resume under **Projects** as:

> **Apollo-Era Core Rope Memory Emulator**
> Designed and built a functional replica of 1960s NASA core rope memory using
> ferrite toroid cores, custom hand-wound sense coils, and analog signal
> conditioning circuits. Implemented electromagnetic induction-based ROM
> readable by an Arduino microcontroller via hardware comparator sensing,
> transistor-switched address lines, and interrupt-driven bit capture.

Add photos of the physical build to this repo — the woven wire assembly is
visually striking and makes the project immediately memorable to anyone
reviewing your GitHub profile.

-----

## 📜 License

MIT License. Build it, fork it, teach with it.

-----

*Inspired by the engineers and weavers at MIT and Raytheon who built the
software that took humans to the Moon — woven one wire at a time.*