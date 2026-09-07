# Hardware Assembly Guide

Wiring, power, and enclosure build for the SafeWay prototype (two-board architecture). Order parts first from [BOM.md](BOM.md).

**Estimated time:** 2–3 hours first build (two boards to wire + the radar verification step)

---

## Tools Required

- Multimeter (continuity + voltage checks)
- Small Phillips screwdriver, drill + 8–10 mm bit (buzzer port / cable glands), file
- Double-sided foam tape / hot glue (module mounting)
- A couple of resistors (10 kΩ + 20 kΩ) — only if the radar's OUT swing needs a divider (§3)
- Optional but ideal for §3: cheap oscilloscope, or 3.5 mm audio cable + laptop mic-input app (free scope apps work fine — Doppler signals are audio-band)

---

## 1. System Overview

A single pole-mounted unit watches the lane. The **CDM324 24 GHz Doppler radar** measures vehicle speed directly from the Doppler frequency shift; the **ESP32-CAM** (on its MB programmer board) photographs the vehicle and serves a live stream; the **ESP32 38-pin hub** counts Doppler pulses, checks the HC-SR04, sounds the buzzer on overspeed, fetches the photo from the CAM, and logs the incident to the cloud API. No cross-road wiring, no second post, no beam alignment.

```
        Road lane  ────────────────────► direction of travel
                      (10–30 m radar coverage)

  ┌ SINGLE POLE UNIT (one IP68 enclosure) ────────────┐
  │                                                    │
  │  ESP32 38-pin HUB                                  │
  │   ├── CDM324 radar OUT ── GPIO 34  (pulse counting)│
  │   ├── HC-SR04 TRIG/ECHO ── GPIO 26/25              │
  │   ├── Buzzer ── GPIO 27                             │
  │   └── WiFi ── fetches CAM photo, POSTs to API       │
  │                                                    │
  │  ESP32-CAM-MB (camera board)                        │
  │   ├── OV2640 /capture  (violation snapshot)         │
  │   ├── /stream  (MJPEG live feed for dashboard)      │
  │   └── microSD: local backup of every photo          │
  │                                                    │
  └────────────────────┬───────────────────────────────┘
                       │ campus WiFi
                       ▼
                  Cloud API + Dashboard
```

**The only physics you need:** the CDM324 outputs a signal whose frequency is speed:

```
f_doppler (Hz) = 44.7 × speed (km/h)
```

A car at 30 km/h → ~1,341 Hz on the OUT pin. The hub counts pulses over a window and divides — that's the whole speed measurement chain.

**Why two boards:** the classic single-board ESP32-CAM starves pins (camera + SD leaves ~2 usable GPIOs — no room for radar, ultrasonic, buzzer). Splitting duties gives the sensors a full 38-pin board and the camera a dedicated board — each simpler to code, flash, and debug; and a camera reboot (heap fragmentation etc.) can never disturb a speed measurement in progress.

---

## 2. The Two Boards

### 2.1 ESP32 38-pin (sensor hub)

Standard DOIT DevKit v1-compatible 38-pin board (CP2102, Type-C or micro-USB). All its GPIOs are free — no camera driver hogging them. SafeWay uses just 3 GPIOs (radar, HC-SR04 ×2 pins, buzzer), leaving 30+ spare for future sensors (PIR, RGB status LED, rain gauge…).

**Board quirks to know:**
- GPIO 34/35/36/39 are **input-only** — perfect for the radar OUT (never needs to drive).
- GPIO 6–11 are flash pins — **never wire anything to them**.
- GPIO 0 is the boot strap — leave it alone or the board won't boot reliably.

**Pin map:**

| GPIO | SafeWay assignment | Notes |
|---|---|---|
| **34** (input-only) | **CDM324 OUT** — Doppler pulses | Input-only pin: the radar only ever drives it; ideal |
| **26** | HC-SR04 TRIG | any OUTPUT-capable GPIO |
| **25** | HC-SR04 ECHO | level-shifted 5 V→3.3 V? see §5.2 note |
| **27** | Buzzer + | any OUTPUT-capable GPIO |
| 32/33 | spare (I2C bus) | future sensors |
| 4/16/17/18/19/21/22/23 | spare (SPI/UART2) | future modules |

### 2.2 ESP32-CAM-MB (camera board)

AI-Thinker-style ESP32-CAM seated on the **MB programmer base board**. The MB board adds: CH340 USB-serial (flash from USB directly — no FTDI, no IO0-to-GND jumper), 5 V power jack, and reset/boot buttons that actually reach the CAM's tiny pads.

**Camera board pin facts:**
- The OV2640 and microSD share GPIOs 0, 5, 13–15 (camera + SD keep ~4 GPIOs busy — that's exactly why it does nothing else).
- Its own pins are all spoken for: **power it, aim it, flash it** — the hub never wires to it.

> **ESP32-CAM + MB compatible-pin note:** the MB board routes the CAM's edge pins (5V, GND, 13, 14, 15) to its own header — meaning if you ever want an emergency fallback (CAM down), the hub could theoretically bit-bang through those. Not used in this project — boards talk over WiFi only.

---

## 3. IF Signal Verification (do this first!)

Before enclosure or firmware, verify what your radar module's OUT pin actually outputs. Seller listings swap boards — spend 10 minutes on the bench:

1. **Power the module:** 5 V to VCC, GND to GND (module silkscreen shows 3–4 pins: VCC, GND, OUT, sometimes a second GND).
2. **Scope the OUT pin** (or feed it into a laptop mic input via a 1 µF coupling cap):
   - **Idle:** near-flat DC, maybe low-frequency noise.
   - **Wave your hand at the module from 1–2 m:** a **rough sine/square wave in the 100–2,000 Hz range** (hand moving ~10–40 km/h equivalent).
   - A fan spinning in front produces a steady tone.
3. **Measure the amplitude:** swing ≤ 3.3 V → wire OUT **directly** to GPIO 34. Swings toward 5 V → add the divider:

```
 CDM324 OUT ──[10 kΩ]──┬──► ESP32 GPIO 34
                       [20 kΩ]
                        │
                       GND
```
(Divides 5 V → 3.3 V. Any nearby 2:1 ratio pair works.)

**If you see only a motion on/off level (flat HIGH when moving)** — the board is a presence-switch variant, not a Doppler-output variant. Return it; the speed design needs the frequency output. (BOM has backups.)

**Record in your build log:** OUT idle level, swing amplitude, waveform at hand-wave. Evidence for the paper's hardware validation section.

---

## 4. Fallback LM358 Signal Conditioner

*Skip if §3 showed healthy pulses.* Repair path if your CDM324 variant's IF is too weak to trigger a digital input (millivolt-level — some bare-sensor boards ship without an amplifier stage despite listing photos):

```
 CDM324 IF ──[C1 10µF]──┬──[R1 10k]── GND          (bias IF at Vcc/2)
                        ├──► LM358 A: non-inv. gain ≈ 100 (R2 1M / R3 10k)
                        │      band-pass ≈ 7 Hz–6 kHz  (0.16–138 km/h)
                        └──► LM358 B: gain ≈ 100      → OUT to GPIO 34
```

- **Easiest:** the prebuilt "LM358 100× Gain Signal Amplification Module" (₱148, BOM) — IF into IN, OUT to GPIO 34, trim idle to mid-rail.
- **Bare IC:** LM358 DIP-8 (₱25/2pcs) on the breadboard, 5 V rail, stages biased at 2.5 V.
- Re-verify with §3's hand-wave test after amplifying — you want pulses swinging 0–3.3+ V.

---

## 5. Wiring Tables

### 5.1 Programming setup (bench only)

| Board | PC |
|---|---|
| ESP32 38-pin USB | USB cable |
| ESP32-CAM-MB USB | USB cable |

Each board flashes itself over its own USB. The CAM-MB's CH340 does the flash dance for you — no IO0 jumper. (Driver notes in [FIRMWARE.md](FIRMWARE.md).)

### 5.2 Main wiring — 38-pin hub

| Module | Pin | Connect to |
|---|---|---|
| CDM324 radar | VCC | 5V rail |
| | GND | GND |
| | OUT | **GPIO 34** (via divider if §3 measured >3.3 V) |
| HC-SR04 | VCC | 5V rail |
| | TRIG | **GPIO 26** |
| | ECHO | **GPIO 25** ← *see level note below* |
| | GND | GND |
| Active buzzer | + | **GPIO 27** |
| | − | GND |

**HC-SR04 ECHO level note:** HC-SR04 outputs 5 V on ECHO. A 5V-tolerant trick used widely: a **10 kΩ + 20 kΩ divider** (same pair as the radar's — buy 2 sets) drops it to 3.3 V. (Some HC-SR04 clones output 3.3 V already; measure once with a multimeter, then decide if the divider's needed.)

### 5.3 Main wiring — CAM board

| Module | Pin | Connect to |
|---|---|---|
| ESP32-CAM-MB | USB (or 5V jack) | its own 5V adapter |
| | microSD | 16 GB FAT32, inserted before power |
| OV2640 camera | built-in ribbon | aimed at the trigger zone |
| (no other wiring — ever) | | |

### 5.4 Power design

```
 Adapter #1 (5V 2A)                Adapter #2 (5V 2A)
        │                                 │
   38-pin HUB                      ESP32-CAM-MB
   ├── CDM324 (5V, ~30–60 mA)         └── (board's onboard reg makes 3.3V)
   ├── HC-SR04 (5V, ~15 mA)
   └── Buzzer (GPIO-driven)
```

- **Separate adapters by design:** WiFi bursts + camera init draw peaks; isolating them means a CAM reboot can't brown-out the hub mid-measurement. (Bench alternative: one adapter + a beefy 1000 µF rail cap, acceptable for testing.)
- Add a 470–1000 µF electrolytic across the hub's 5V rail.
- **Radar stability:** keep a cap near the radar's VCC/GND pair so WiFi bursts don't modulate the radar supply (phantom low-speed readings).

---

## 6. microSD Preparation (CAM board)

1. Format **FAT32** (Windows: right-click drive → Format → FAT32, 32 KB clusters).
2. Class 10 / U1 minimum — capture stalls below this.
3. Insert **before** powering (hot-insert is unreliable).
4. 16 GB holds ~40,000 JPEGs at SVGA — weekly rotation is plenty.

---

## 7. Enclosure Assembly (IP68 ABS, 200x100x70 mm)

1. **Layout:** 830-point breadboard + 38-pin board (hub) on one side; CAM-MB on the other; radar module centered behind the front wall; terminal strip for power in/out.
2. **Radar mounting:** 24 GHz passes through **ABS plastic** (not metal). Mount the CDM324 **inside** the sealed box facing out through the plastic wall — zero apertures for rain. Antenna face within ~2 cm of the wall.
   - Never put metal (screws, brackets, foil) between radar and road.
3. **HC-SR04 placement:** the ultrasonic transducer must see outdoors — drill two 16 mm holes (transducer + receiver barrels) or mount it just inside a cutout with silicone seal. Aim it at the **trigger zone** (the road point nearest the pole — where the vehicle is when the photo is taken).
4. **Buzzer:** small drilled port (8 mm) covered with tape.
5. **Camera window:** large cutout + clear acrylic/PETG sealed with silicone. OV2640 must see the lane at plate height.
6. **Cable glands:** one for DC power in, one spare for future sensor. Glue-lined heat shrink.
7. **Desiccant pack** inside — tropical humidity fogs lenses.
8. Breadboard sticks down with double-sided foam tape (removable for iteration).

---

## 8. Radar Aiming Procedure (replaces gate alignment)

1. **Mount the pole unit** so the radar beam runs **along the traffic direction**, angled **≤15° off the lane axis** (10–15° sweet spot: enough angle to not be a head-on reflector, small enough that cosine error stays under 4%).
2. **Cosine error, the one number to know:** radar measures speed along the beam — Measured = true × cos(θ):

| Aiming angle θ off traffic axis | Reading error |
|---|---|
| 0° | 0% |
| 10° | −1.5% |
| 15° | −3.4% |
| 30° | −13.4% (re-aim) |

   The firmware's `COSINE_ANGLE_DEG` constant compensates; keep θ small or set the measured install angle.
3. **Field of view:** clear the beam corridor (a ~15–30° cone) of **swaying branches, banners, AC condenser fans, parked vehicles** — anything moving inside the cone reads as a target.
4. **HC-SR04 aims at the trigger zone** (near zone, 2–4 m ahead of the pole at plate height): distance suddenly dropping = vehicle physically present → confirms radar event, rejects phantom triggers.
5. **Bench-verify first:** serial shows Hz values on a hand-wave 1–3 m out, ~0 Hz when still. Then graduate to vehicles ([TESTING.md](TESTING.md)).
6. Re-check aim after typhoons — a knocked-5° pole quietly adds cosine error to every record.

---

## 9. Assembly Checklist

- [ ] §3 IF verification done — waveform confirmed, amplitude recorded, divider fitted if needed
- [ ] Hub pins: radar→34, HC-SR04→26/25 (ECHO divider if needed), buzzer→27
- [ ] Both boards power up independently on their own adapters
- [ ] CAM board: microSD FAT32 in before power; camera focused on trigger zone
- [ ] 5V rails measure 4.8–5.2 V under WiFi load; cap installed near the radar
- [ ] Radar faces out through ABS wall, no metal in the beam corridor
- [ ] HC-SR04 transducers outdoors via sealed cutouts, aimed at the trigger zone
- [ ] Camera window clear, desiccant in, glands sealed
- [ ] Serial: ~0 Hz idle noise floor, Hz spikes on hand movement, HC-SR04 distance sane (±3 cm on a wall)

---

Build chain: [BOM](BOM.md) → **HARDWARE** → [FIRMWARE](FIRMWARE.md) → [BACKEND](BACKEND.md) → [TESTING](TESTING.md) → [DEPLOYMENT](DEPLOYMENT.md)
