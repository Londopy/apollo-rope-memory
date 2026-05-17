# 🧵 Core Rope Memory — Arduino Replica

A functional replica of 1960s NASA-era core rope memory, the read-only storage
technology that flew humans to the Moon aboard the Apollo Guidance Computer.
This project encodes binary data physically by weaving wires through ferrite
toroid cores and reads it back using electromagnetic induction, an op-amp
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
small voltage spike in any core it passes through. A sense coil wound around
each core picks up this spike. An op-amp comparator amplifies it to a 5V
logic level that the Arduino can read.

Cores where the wire was routed around the outside see no induction — they
read as 0. This is entirely passive ROM: no power needed to retain data.

-----

## 🛠️ Parts List

|Part                |Spec / Part Number                  |Qty|Source |Est. Cost|
|--------------------|------------------------------------|---|-------|---------|
|Arduino Uno         |Rev3                                |1  |Amazon |$10.00   |
|Ferrite toroid cores|Fair-Rite 2643102702 (~7mm OD, MnZn)|10 |DigiKey|$3.00    |
|Enameled magnet wire|30 AWG, 1 oz spool                  |1  |Amazon |$8.00    |
|Op-amp              |LM358, DIP-8 package                |10 |DigiKey|$5.00    |
|NPN transistor      |2N2222, TO-92 package               |20 |DigiKey|$2.00    |
|Signal diode        |1N4148, DO-35                       |20 |DigiKey|$1.00    |
|Resistor assortment |100Ω, 1kΩ, 10kΩ — 1/4W              |—  |Amazon |$5.00    |
|Breadboard          |830 tie-point                       |1  |Amazon |$6.00    |
|Jumper wire kit     |M-M assortment                      |1  |Amazon |$5.00    |

**Total: ~$35–40**

### Optional but Highly Recommended

|Part                    |Notes                                           |Est. Cost|
|------------------------|------------------------------------------------|---------|
|DSO138 Oscilloscope Kit |Essential for debugging the analog sense circuit|~$15     |
|Helping hands / PCB vice|Holds cores while winding sense coils           |~$8      |
|Wire stripper / tweezers|For magnet wire work                            |~$5      |

-----

## ⚡ Circuit Overview

### Full Signal Chain

```
[ Arduino Digital Out ]
          │
          ▼
  [ 1kΩ Base Resistor ]
          │
          ▼
  [ 2N2222 Transistor ]   ← Collector to Word Wire, Emitter to GND
          │                  1N4148 flyback diode across C-E (cathode → Vcc)
          ▼
  [ Word Wire ]
  ├── Through Core 1  (Bit 7 = 1)
  ├── Around  Core 2  (Bit 6 = 0)
  ├── Through Core 3  (Bit 5 = 1)
  └── ...
          │
          ▼
        [ GND ]


[ Sense Coil on Core N ]  (15–20 turns, 30 AWG)
          │
          ▼
  [ LM358 Op-Amp Comparator ]
  ├── (+) input: sense coil output
  └── (−) input: ~150mV reference voltage (voltage divider from 5V)
          │
          ▼
  [ Arduino Digital Input Pin ]
```

### Op-Amp Comparator Reference Voltage

Use a voltage divider to set the comparator threshold:

```
5V ──[ 33kΩ ]──┬──[ 220kΩ ]── GND
               │
           (−) input of LM358
           (~150 mV)
```

Adjust the divider ratio if you’re getting false triggers (lower threshold)
or missing real spikes (raise threshold). An oscilloscope helps enormously here.

-----

## 🔧 Step-by-Step Build Guide

### Step 1 — Wind the Sense Coils

1. Cut 8 lengths of 30 AWG magnet wire, each about 30 cm long.
1. Using tweezers or a needle, wind **15–20 tight turns** through the center
   hole of each ferrite toroid core.
1. Leave ~5 cm of wire on each end as leads.
1. Scrape the enamel off the last 5 mm of each lead using fine sandpaper or
   a lighter flame, then tin with solder.
1. Label each core 0–7 with a marker or small sticker.

> **Tip:** More turns = stronger induced signal = easier to detect.
> 20 turns is better than 10.

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

Take a length of magnet wire and route it through or around each core
in sequence according to your data. This wire is your **word wire** —
it represents one memory address.

To store multiple bytes, weave additional independent word wires through
the same 8 cores, each encoding a different byte.

-----

### Step 3 — Build the Transistor Driver

For each word wire:

1. Connect the word wire’s input end to the **Collector** of a 2N2222.
1. Connect the **Emitter** to GND.
1. Connect the **Base** to an Arduino digital output pin via a 1kΩ resistor.
1. Place a 1N4148 diode across Collector–Emitter with the cathode toward
   the word wire (flyback protection).
1. Connect the far end of the word wire to +5V or a current-limiting resistor
   (~100Ω) to 5V.

-----

### Step 4 — Build the Op-Amp Sense Circuit

For each of the 8 cores:

1. Connect one end of the sense coil to GND.
1. Connect the other end to the **(+) non-inverting input** of one LM358 channel.
1. Build a voltage divider (33kΩ / 220kΩ from 5V to GND) and connect the
   midpoint to the **(−) inverting input** of the same LM358 channel.
1. Connect the LM358 **output** to an Arduino digital input pin.
1. Add a 10kΩ pull-down resistor from the LM358 output to GND.

The LM358 has two op-amp channels per chip, so you need 4 chips for 8 cores.

-----

### Step 5 — Wire Arduino Pins

Suggested pin mapping:

```
Word Wire 0 (address 0x00) → Arduino D2  (via transistor)
Sense Core 0 (Bit 7)       → Arduino D3
Sense Core 1 (Bit 6)       → Arduino D4
Sense Core 2 (Bit 5)       → Arduino D5
Sense Core 3 (Bit 4)       → Arduino D6
Sense Core 4 (Bit 3)       → Arduino D7
Sense Core 5 (Bit 2)       → Arduino D8
Sense Core 6 (Bit 1)       → Arduino D9
Sense Core 7 (Bit 0)       → Arduino D10
```

-----

### Step 6 — Upload and Test

Upload the Arduino sketch (see below), open the Serial Monitor at 9600 baud,
and confirm the byte you wove matches what is printed.

-----

## 💻 Arduino Code

```cpp
// ============================================================
// Core Rope Memory Reader
// 1-byte (8-bit) example
// ============================================================

// --- Pin Definitions ---
const int WORD_WIRE_PIN   = 2;
const int SENSE_PINS[8]   = {3, 4, 5, 6, 7, 8, 9, 10};

// --- Timing (microseconds) ---
const int PULSE_US        = 500;
const int SETTLE_US       = 100;

// ============================================================
void setup() {
  Serial.begin(9600);
  pinMode(WORD_WIRE_PIN, OUTPUT);
  digitalWrite(WORD_WIRE_PIN, LOW);

  for (int i = 0; i < 8; i++) {
    pinMode(SENSE_PINS[i], INPUT);
  }

  Serial.println("================================================");
  Serial.println("  Core Rope Memory Reader — Ready");
  Serial.println("================================================");
}

// ============================================================
byte readWordWire() {
  byte result = 0;

  // 1. Pulse the word wire
  digitalWrite(WORD_WIRE_PIN, HIGH);
  delayMicroseconds(PULSE_US);
  digitalWrite(WORD_WIRE_PIN, LOW);

  // 2. Allow op-amp outputs to settle
  delayMicroseconds(SETTLE_US);

  // 3. Sample all 8 sense pins
  for (int i = 0; i < 8; i++) {
    if (digitalRead(SENSE_PINS[i]) == HIGH) {
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
    if (i == 4) Serial.print(" ");   // space in middle for readability
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

To change the stored data, you physically re-weave the word wire:

- **Store a 1** → thread the wire through the center hole of the core
- **Store a 0** → route the wire around the outside of the core

To store multiple bytes, add a second transistor on a new Arduino output pin
and weave a second word wire through the same 8 cores. Extend the sketch to
pulse each address line separately and print each result.

-----

## 🐛 Troubleshooting

|Symptom                     |Likely Cause                        |Fix                                                         |
|----------------------------|------------------------------------|------------------------------------------------------------|
|All bits read as 0          |No induction signal reaching Arduino|Check sense coil connections; verify op-amp wiring          |
|All bits read as 1          |Comparator threshold too low        |Raise the reference voltage divider                         |
|Random / noisy output       |Electrical noise on breadboard      |Add 100nF decoupling caps across LM358 Vcc pins             |
|Correct pattern, wrong value|Bit ordering reversed               |Swap bit index in firmware: `(1 << i)` vs `(1 << (7-i))`    |
|Pulse fires but no response |Transistor not switching            |Check base resistor; verify 2N2222 pinout (EBC order varies)|


> An oscilloscope on the sense coil output (before the op-amp) should show a
> clean spike of a few millivolts when a pulse is sent. If you see nothing,
> the issue is mechanical (coil winding or wire routing). If you see a spike
> but the Arduino doesn’t register it, the issue is in the comparator circuit.

-----

## 📄 Resume / Portfolio Notes

This project can be listed on a resume under **Projects** as:

> **Apollo-Era Core Rope Memory Emulator**
> Designed and built a functional replica of 1960s NASA core rope memory using
> ferrite toroid cores, custom hand-wound sense coils, and analog signal
> conditioning circuits. Implemented electromagnetic induction-based ROM
> readable by an Arduino microcontroller via op-amp comparator sensing and
> transistor-switched address lines.

Add photos of the physical build to this repo — the woven wire assembly is
visually striking and makes the project immediately memorable to anyone
reviewing your GitHub profile.

-----

## 📜 License

MIT License. Build it, fork it, teach with it.

-----

*Inspired by the engineers and weavers at MIT and Raytheon who built the
software that took humans to the Moon — woven one wire at a time.*