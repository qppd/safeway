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

A single pole-mounted unit watches the lane. The **CDM324 24 GHz Doppler radar** measures vehicle speed directly from the Doppler frequency shift; the **ESP32-S3 WROOM N16R8 CAM board** (OV5640) photographs the vehicle and serves a live stream; the **ESP32 38-pin hub** counts Doppler pulses, watches the **laser break-beam** (KY-008 transmitter on a far post, receiver module on the hub pole — a vehicle crossing the lane breaks the beam), sounds the buzzer on overspeed, snapshots the photo at the beam-break moment, and logs the incident to the cloud API when the lane clears. No data cables across the road — just one thin 2-wire power run to the far-post laser.

```
        Road lane  ────────────────────► direction of travel
                      (10–30 m radar coverage)

   FAR POST                      HUB POLE
  ┌ KY-008 laser TX ─────╳╳╳╳╳──── laser receiver ─┐
  │ (red dot, always-on)  beam        (module, DO) │
  │ powered by 22AWG 2-wire run ────► ESP32 38-pin HUB
  │                                     ├── CDM324 radar OUT ── GPIO 34
  │                                     ├── Laser receiver DO ── GPIO 25
  │                                     ├── Buzzer ── GPIO 27
  │                                     └── WiFi ── fetches CAM photo, POSTs to API
  │                                     ESP32-S3 WROOM CAM
  │                                     ├── OV5640 /capture  (violation snapshot)
  │                                     ├── /stream  (polled live frame for dashboard)
  │                                     └── microSD: local backup of every photo
  └────────────────────┬────────────────────────────────┘
                       │ campus WiFi
                       ▼
                  Cloud API + Dashboard
```

**The only physics you need:** the CDM324 outputs a signal whose frequency is speed:

```
f_doppler (Hz) = 44.7 × speed (km/h)
```

A car at 30 km/h → ~1,341 Hz on the OUT pin. The hub counts pulses over a window and divides — that's the whole speed measurement chain.

**Why two boards:** the classic single-board ESP32-CAM starves pins (camera + SD leaves ~2 usable GPIOs — no room for radar, break-beam, buzzer). The new ESP32-S3 camera board has GPIOs to spare, but the split stays: radar pulse-counting never competes with camera DMA for the same core, and a camera reboot (heap fragmentation etc.) can never disturb a speed measurement in progress. Each board is simpler to code, flash, and debug on its own.

---

## 2. The Two Boards

### 2.1 ESP32 38-pin (sensor hub)

Standard DOIT DevKit v1-compatible 38-pin board (CP2102, Type-C or micro-USB). All its GPIOs are free — no camera driver hogging them. SafeWay uses just 3 GPIOs (radar, beam receiver, buzzer), leaving 30+ spare for future sensors (PIR, RGB status LED, rain gauge…).

**Board quirks to know:**
- GPIO 34/35/36/39 are **input-only** — perfect for the radar OUT (never needs to drive).
- GPIO 6–11 are flash pins — **never wire anything to them**.
- GPIO 0 is the boot strap — leave it alone or the board won't boot reliably.

**Pin map:**

| GPIO | SafeWay assignment | Notes |
|---|---|---|
| **34** (input-only) | **CDM324 OUT** — Doppler pulses | Input-only pin: the radar only ever drives it; ideal |
| **25** | **Laser receiver DO** (beam broken/OK) | digital in; receiver is a 3.3 V-safe comparator output — see §5.2 note |
| **27** | Buzzer + | any OUTPUT-capable GPIO |
| 32/33 | spare (I2C bus) | future sensors |
| 4/16/17/18/19/21/22/23/26 | spare (SPI/UART2 etc.) | future modules |

**Boot & WiFi safety audit (why these exact pins):**

| GPIO | Pin class on ESP32-WROOM-32 | Safe for our use? |
|---|---|---|
| 34 | Input-only — no strap role, no WiFi role | ✅ radar OUT drives it push-pull |
| 25 | ADC2 channel / DAC1 | ✅ digital beam input only (comparator DO) |
| 27 | ADC2 channel / touch T7 | ✅ digital buzzer output |

- The "GPIO 25/26/27 conflict with WiFi" rule is real but applies **only to `analogRead()`** — the WiFi driver owns the ADC2 peripheral, so analog reads on those pins fail while WiFi is active. **Digital I/O and interrupts are unaffected**; this firmware never calls analog functions.
- **Strapping pins 0, 2, 5, 12, 15 are entirely avoided** — nothing can pull the boot mode, change flash voltage (the classic GPIO-12 killer), or silence boot logs.
- Flash pins **6–11** and UART0 **1/3** untouched.
- **GPIO 34 has no internal pull-up/down** (true for all of 34–39): the CDM324's amplified OUT drives it push-pull so nothing is needed — but if the signal ever reads flaky, add an external **10 kΩ pull-up to 3.3 V**. Don't rely on `INPUT_PULLUP` on 34–39; it silently does nothing.
- **Laser receiver on GPIO 25 (has internal pull-ups):** the receiver's open-comparator DO idles at a defined level with `INPUT_PULLUP` enabled — no external resistor needed.
- Interrupt load is trivial: the radar ISR is a single increment (≤ ~4,470 pulses/s at 100 km/h); the beam ISR fires only on state changes (a car passes = 2 edges), debounced in software.

### 2.2 ESP32-S3 WROOM N16R8 CAM (camera board)

Standalone ESP32-S3-WROOM-1 **N16R8** board (16 MB flash, 8 MB octal PSRAM) with the **OV5640 5 MP camera** on a 24-pin DVP FPC header and an **onboard microSD slot**. No programmer board, no FTDI, no IO0 jumper — two Type-C ports (one CH343P USB-serial for flashing, one native USB-OTG).

**Camera board pin facts** (from the board's pinout diagram, cross-checked against the xiaozhi-esp32 `bread-compact-wifi-s3cam` config — 14/14 camera pins match):
- Camera DVP bus: Y2–Y9 = GPIO 11, 9, 8, 10, 12, 18, 17, 16 · XCLK 15 · PCLK 13 · VSYNC 6 · HREF 7 · SIOD 4 · SIOC 5 · PWDN/RESET not wired.
- microSD (1-bit SD_MMC): CLK = GPIO 39 · CMD = GPIO 38 · D0 = GPIO 40.
- The 24-pin header accepts OV2640/OV7725/OV3660/OV5640 — OV5640 is the fitted 5 MP unit.
- Still dedicated in this project: **power it, aim it, flash it** — the hub never wires to it. Spare GPIOs (19, 20, 21, 35–37, 45, 48…) stay free for future on-board peripherals.

> **3.3 V logic only:** the ESP32-S3 GPIOs are **not 5 V tolerant** (unlike the classic WROOM-32 hub, which survives it). Never feed 5 V into any S3 pin — including its camera/SD lines.

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
| ESP32-S3 CAM USB-C (either port) | USB cable |

Each board flashes itself over its own USB. The CAM board's CH343P does the flash dance for you — no IO0 jumper. (Driver + Arduino board-settings notes in [FIRMWARE.md](FIRMWARE.md).)

### 5.2 Main wiring — 38-pin hub + far post

| Module | Pin | Connect to |
|---|---|---|
| CDM324 radar | VCC | 5V rail |
| | GND | GND |
| | OUT | **GPIO 34** (via divider if §3 measured >3.3 V) |
| **Laser receiver** (hub pole) | VCC | 5V rail (or 3.3 V — see note) |
| | GND | GND |
| | DO | **GPIO 25** (`INPUT_PULLUP`) |
| | AO (if present) | leave unconnected |
| **KY-008 laser TX** (far post) | S | 5V rail *(always-on — no hub pin needed)* |
| | middle (+) | 5V rail |
| | − | GND |
| | — | both rails come down the 22AWG 2-wire run from the hub enclosure |
| Active buzzer | + | **GPIO 27** |
| | − | GND |

**Laser receiver level note:** most laser-receiver modules in this family (the "Non-Modulator Tube Laser Receiver" / KY-008-pair type) run a 5 V supply but output a **3.3 V-safe digital DO** from an LM393-style comparator — still, **measure DO once with a multimeter** (beam open vs blocked) before trusting it: if it swings to ~5 V, add the same **10 kΩ + 20 kΩ divider** as the radar's. If your module has an **AO (analog) pin**, ignore it — we only use DO.

**Beam polarity — check once, code-free:** power the pair on the bench, aim TX at receiver, and watch the receiver board's onboard LED (most have one): typically **LED ON / DO LOW = beam intact**, **LED OFF / DO HIGH = beam broken**. The firmware's `BEAM_BREAKS_LOW` constant matches whichever polarity your module uses — set it after this bench check ([FIRMWARE.md §5](FIRMWARE.md#5-tuning-constants)).

**Far-post laser mounting:** KY-008 TX in a small weatherproof box or under an overhang, dot aimed across the lane at the receiver's photodetector. Mount both ends at the same height (plate height is ideal: ~50 cm) so the beam crosses where plates are. Keep the dot small — at ≤ 10 m a 6 mm copper-head module holds a tight dot without optics.

### 5.3 Main wiring — CAM board

| Module | Pin | Connect to |
|---|---|---|
| ESP32-S3 WROOM CAM | Type-C USB (either port) or 5V/VIN pin | its own 5V adapter |
| | microSD (onboard slot) | 16 GB FAT32, inserted before power |
| OV5640 camera | 24-pin FPC header (fitted) | aimed at the trigger zone |
| (no other wiring — ever) | | |

### 5.4 Power design

```
 Adapter #1 (5V 2A)                       Adapter #2 (5V 2A)
        │                                       │
   38-pin HUB                               ESP32-S3 WROOM CAM
   ├── CDM324 (5V, ~30–60 mA)                    └── (board's onboard reg makes 3.3V)
   ├── Laser receiver (5V, ~5 mA)
   ├── Buzzer (GPIO-driven)
   └── 22AWG 2-wire run ──► FAR POST: KY-008 TX (5V, <30 mA)
```

- **Separate adapters by design:** WiFi bursts + camera init draw peaks; isolating them means a CAM reboot can't brown-out the hub mid-measurement. (Bench alternative: one adapter + a beefy 1000 µF rail cap, acceptable for testing.)
- Add a 470–1000 µF electrolytic across the hub's 5V rail.
- **Radar stability:** keep a cap near the radar's VCC/GND pair so WiFi bursts don't modulate the radar supply (phantom low-speed readings).
- **Far-post run:** < 30 mA over 22AWG is millivolts of drop — the laser runs full-brightness even at 10 m of cable. Fuse the run (or use an adapter with built-in protection) so a nicked cable can't short the rail.

---

## 6. microSD Preparation (CAM board)

1. Format **FAT32** (Windows: right-click drive → Format → FAT32, 32 KB clusters).
2. Class 10 / U1 minimum — capture stalls below this.
3. Insert **before** powering (hot-insert is unreliable).
4. 16 GB holds ~40,000 JPEGs at SVGA — weekly rotation is plenty.

---

## 7. Enclosure Assembly (IP68 ABS, 200x100x70 mm)

1. **Layout:** 830-point breadboard + 38-pin board (hub) on one side; ESP32-S3 CAM board on the other; radar module centered behind the front wall; terminal strip for power in/out.
2. **Radar mounting:** 24 GHz passes through **ABS plastic** (not metal). Mount the CDM324 **inside** the sealed box facing out through the plastic wall — zero apertures for rain. Antenna face within ~2 cm of the wall.
   - Never put metal (screws, brackets, foil) between radar and road.
3. **Laser receiver placement:** mount it inside the enclosure behind a small clear window (drill ~10–12 mm, seal with silicone or a glue-lined washer) — the photodetector just needs to see the far-post dot. Aim the window at the **beam axis** (straight across the lane).
4. **Far-post KY-008 TX:** in its own small weatherproof housing (or tucked under the post cap), dot aimed at the receiver window. The 22AWG run leaves the hub enclosure through a cable gland, follows the curb/ground, and enters the far post's housing — UV ties every 30 cm; conduit sleeve where cars might roll over it.
5. **Buzzer:** small drilled port (8 mm) covered with tape.
6. **Camera window:** large cutout + clear acrylic/PETG sealed with silicone. OV5640 must see the lane at plate height.
7. **Cable glands:** one for DC power in, one for the far-post laser run, one spare for future sensor. Glue-lined heat shrink.
8. **Desiccant pack** inside — tropical humidity fogs lenses.
9. Breadboard sticks down with double-sided foam tape (removable for iteration).

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
4. **Beam alignment (one-time, two-person):** one person holds a white card at the receiver window; the other nudges the far-post KY-008 until the red dot lands on the receiver's photodetector (receiver LED flips → dot is on target). At ≤ 10 m the dot barely diverges — align once at install and it holds. Re-check after typhoons (a knocked far post is the #1 beam failure mode).
5. **Bench-verify first:** serial shows Hz values on a hand-wave 1–3 m out, ~0 Hz when still. Then graduate to vehicles ([TESTING.md](TESTING.md)).
6. Re-check aim after typhoons — a knocked-5° pole quietly adds cosine error to every record.

---

## 9. Assembly Checklist

- [ ] §3 IF verification done — waveform confirmed, amplitude recorded, divider fitted if needed
- [ ] Hub pins: radar→34, laser receiver DO→25 (divider only if measured >3.3 V), buzzer→27
- [ ] KY-008 bench check: TX dot visible, receiver LED flips when beam blocked, DO polarity recorded
- [ ] Both boards power up independently on their own adapters
- [ ] CAM board: microSD FAT32 in before power; camera focused on trigger zone
- [ ] 5V rails measure 4.8–5.2 V under WiFi load; cap installed near the radar
- [ ] Radar faces out through ABS wall, no metal in the beam corridor
- [ ] Receiver window drilled + sealed; far-post TX housed; 22AWG run tied down with drip loops
- [ ] Camera window clear, desiccant in, glands sealed
- [ ] Serial: ~0 Hz idle noise floor, Hz spikes on hand movement; beam blocked → hub prints `BEAM BROKEN`; beam blocked mid-event → `SNAPSHOT: ok` within ~1 s

---

Build chain: [BOM](BOM.md) → **HARDWARE** → [FIRMWARE](FIRMWARE.md) → [BACKEND](BACKEND.md) → [TESTING](TESTING.md) → [DEPLOYMENT](DEPLOYMENT.md)
