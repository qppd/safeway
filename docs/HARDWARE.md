# Hardware Assembly Guide

Wiring, power, and enclosure build for the SafeWay prototype. Order parts first from [BOM.md](BOM.md).

**Estimated time:** 2–3 hours (first build)

---

## Tools Required

- Soldering iron + solder (only if using the bare SOT-223 AMS1117; module version needs none)
- Multimeter (continuity + voltage checks)
- Small Phillips screwdriver, drill + 8–10 mm bit (laser apertures in enclosure), file
- Double-sided foam tape / hot glue for breadboard mounting
- Ruler/tape measure (gate separation must be measured precisely — it feeds the speed formula)

---

## System Overview

Two **break-beam gates** sit 2–3 m apart along the monitored lane. Each gate = one **KY-008 laser transmitter** aimed at one **laser receiver module**. The receivers' digital outputs interrupt the ESP32-CAM, which timestamps each beam break and computes speed.

```
        Road lane  ────────────────────► direction of travel

  ┌ GATE A ─────────┐        ┌ GATE B ─────────┐
  │ KY-008 ►►► beam │ 2–3 m  │ KY-008 ►►► beam │
  │            ▼    │        │            ▼    │
  │     Receiver OUT │        │     Receiver OUT │
  └───────┬──────────┘        └───────┬─────────┘
          │ GPIO 13                   │ GPIO 12
          └────────────┬──────────────┘
                       ▼
               ESP32-CAM + microSD + buzzer (GPIO 15)
                       │ WiFi
                       ▼
                  Cloud API
```

---

## ESP32-CAM Pin Map (what's usable and why)

The AI-Thinker ESP32-CAM has very few free GPIOs — the camera and microSD claim most of them. This design uses the microSD in **1-bit mode** (set in firmware), which releases **GPIO 12 and 13** for the sensors:

| GPIO | Used by | SafeWay assignment |
|---|---|---|
| 12 | SD DAT1 (freed in 1-bit mode) | **Gate B receiver OUT** |
| 13 | SD DAT2 (freed in 1-bit mode) | **Gate A receiver OUT** |
| 15 | SD DAT3 (pull-up, still usable) | **Buzzer +** |
| 14 | SD CLK | leave for SD |
| 2, 4 | SD CMD/DATA0 | leave for SD (GPIO 4 = onboard flash LED) |
| 16 | PSRAM CS | **do not use** |
| 0 | boot strapping | jumper to GND only when flashing |
| 1 / 3 | UART TX/RX | serial monitor / debug |

> Note: If you skip the microSD entirely, more pins free up — but the paper's design logs photos locally as backup, so keep it.

---

## Wiring Tables

### 1. Programming setup (temporary, on the bench)

| CH340 USB-TTL | ESP32-CAM |
|---|---|
| 5V | 5V |
| GND | GND |
| TX | U0R (GPIO 3) |
| RX | U0T (GPIO 1) |
| — | **GPIO 0 ↔ GND** (jumper only while flashing — remove to run) |

Set the CH340 jumper to **5V** for flashing. Details: [FIRMWARE.md](FIRMWARE.md#flashing-procedure).

### 2. Laser gates

| Module | Pin | Connect to |
|---|---|---|
| KY-008 transmitter (×2) | S | (leave unconnected — always-on beam) |
| | + (middle) | 5V rail |
| | − | GND |
| Receiver A (Gate A) | VCC | 3.3V rail (AMS1117 output) |
| | GND | GND |
| | OUT/DO | **GPIO 13** |
| Receiver B (Gate B) | VCC | 3.3V rail |
| | GND | GND |
| | OUT/DO | **GPIO 12** |
| Buzzer (+) | — | **GPIO 15** |
| Buzzer (−) | — | GND |

**Receiver polarity check (do this first!):** Before connecting to the ESP32-CAM, power one receiver and measure OUT with a multimeter — note the reading with the laser shining on it vs. blocked. Most modules output **HIGH when beam is blocked** (beam received = LOW), but verify yours — the firmware has an `BEAM_BLOCKED_STATE` constant to match either behavior.

**KY-008 note:** the KY-008's `S` pin can switch the laser on/off (HIGH = on for most modules). Leaving S unconnected and powering from 5V gives a continuous beam — simplest and most reliable for break-beam timing. Continuous draw is ~30 mA per transmitter.

### 3. Power design

```
 5V 2A adapter (DC jack)
   ├── ESP32-CAM 5V pin  (onboard regulator makes its own 3.3V)
   ├── KY-008 × 2         (5V, always on)
   ├── Buzzer             (5V-tolerant, driven from GPIO 15)
   └── AMS1117 3.3V IN
          └── 3.3V OUT ──► laser receivers (×2)
```

- The ESP32-CAM + WiFi bursts need headroom — that's why the BOM specifies **2A**, not 1A.
- AMS1117 dropout is ~1.1 V, so 5 V in → ~3.9 V max out... in practice the module regulates fine to 3.3 V; if you measure under 3.2 V under load, feed it from the 5V rail directly (input range 4.5–12 V).
- Add a 470 µF–1000 µF electrolytic cap across the 5V rail inside the enclosure — WiFi transmit bursts cause brownouts otherwise (a very common ESP32-CAM failure mode).

---

## microSD Preparation

1. Format as **FAT32** (Windows: right-click drive → Format → FAT32; use 32 KB allocation size).
2. Class 10 / U1 minimum — photo capture stalls below this.
3. Insert **before** powering the ESP32-CAM (hot-insert is unreliable).
4. 16 GB holds roughly 40,000+ JPEGs — weekly rotation is plenty.

---

## Enclosure Assembly (IP68 ABS box, 200×100×70 mm)

1. **Plan the layout first** on paper: breadboard (165×55 mm), ESP32-CAM, AMS1117, terminal strip for gate wiring. The 200×100 box fits all of it with room for the buzzer.
2. **Laser apertures:** the two KY-008 transmitters and receivers ideally mount *outside* the box facing each other across the lane (each gate = transmitter on one side, receiver on the opposite side of the road/lane). If mounting everything in one box instead (bench/POC), drill two 8–10 mm holes for the beams and seal with hot glue around the module flanges.
3. **Cable glands / knockouts:** one for the DC power input, one per gate's 3-wire run to the receiver/transmitter. IP68 rating is defeated by unsealed holes — use glue-lined heat shrink or silicone sealant.
4. **Mount breadboard** with double-sided foam tape (it's removable for iteration).
5. **Buzzer outside or at a vent hole** — sound is muffled inside a sealed ABS box. A small hole over the buzzer port restores loudness; cover with tape to keep water out.
6. **Desiccant pack** inside — tropical humidity will fog the camera lens otherwise.
7. **Camera window:** the OV2640 must see the lane. Best: large cutout with a clear acrylic/PETG window sealed with silicone. Acceptable for POC: camera positioned at an open lid edge during testing.

---

## Gate Alignment Procedure

1. Mount the two transmitters at the same height (~0.8–1.2 m — car bumper/plate height) on posts/brackets 2–3 m apart along the lane, both perpendicular to travel.
2. Aim each transmitter at its receiver across the lane. The receiver's onboard LED (most modules have one) lights when the beam lands — use it for coarse aim.
3. Fine-tune until the LED is stable; then verify with the multimeter that OUT toggles when you wave a hand through the beam.
4. **Measure the exact center-to-center distance between Gate A and Gate B** with a tape measure. This value goes into `GATE_DISTANCE_M` in the firmware — the speed math is only as accurate as this measurement.
5. Re-check alignment after any wind/rain — the #1 field failure is beam drift.

---

## Assembly Checklist

- [ ] Both receiver OUTs verified (beam-blocked voltage level noted)
- [ ] GPIO 12 / 13 / 15 assignments match the firmware
- [ ] 5V rail measures 4.8–5.2 V under WiFi load
- [ ] AMS1117 output measures 3.2–3.3 V
- [ ] microSD inserted, FAT32-formatted
- [ ] Enclosure sealed: glands tight, desiccant in, camera window clear
- [ ] Gate separation measured and written down
- [ ] Flash jumper (GPIO 0 ↔ GND) **removed** before field run

Next: flash the firmware → [FIRMWARE.md](FIRMWARE.md)
