# SafeWay — Bill of Materials (BOM)

**Project:** IoT Vehicle Speed Monitoring System (CDM324 24 GHz Doppler radar + laser break-beam confirm + two-board ESP32 architecture)
**Source:** Lazada Philippines · Prices checked **Sep 7–8, 2026**
**Criteria:** 3+ rating where available, proven sales (sold count), local PH sellers preferred, cheapest that qualifies.
**Note:** Lazada prices move with vouchers/promos — treat totals as estimates (±10%).

---

## Architecture (drives this BOM)

```
  ┌─ ESP32 38-pin HUB (sensor board) ──────────────────┐
  │  CDM324 radar OUT ──► GPIO (Doppler pulse counting) │
  │  Laser receiver DO ──► GPIO 25 (presence confirm)   │
  │  Buzzer ──► GPIO (overspeed alert)                   │
  │  WiFi: fetches photo from CAM, POSTs to cloud API    │
  └──────────────┬──────────────────────────────────────┘
                 │ same WiFi network — zero wires between boards
  ┌─ ESP32-CAM-MB (camera board) ─────────────────────┐
  │  OV2640 photo + live stream server                  │
  │  microSD: local backup of every violation photo      │
  │  MB programmer board: CH340 = easy USB flashing      │
  └─────────────────────────────────────────────────────┘

  ┌─ FAR POST (opposite side of the lane) ─────────────┐
  │  KY-008 laser TX, always-on (S strapped to VCC)     │
  │  powered by a 2-wire 22AWG run from the hub box     │
  │  red dot crosses the lane to the receiver           │
  └─────────────────────────────────────────────────────┘
```

**Why two boards:** the classic AI-Thinker ESP32-CAM starves you for pins (camera + SD leaves ~2–3 usable GPIOs — no room for radar, beam receiver, and buzzer). Splitting duties gives the sensors a full 38-pin board and the camera a dedicated board — each simpler to code, flash, and debug. They meet over WiFi; no wires between them (the far-post laser TX is the only cable in the system — two thin wires).

---

## Summary Table

| # | Component | Qty | Unit Price | Subtotal | Rating | Sold |
|---|-----------|-----|-----------:|---------:|--------|------|
| 1 | CDM324 24 GHz Doppler radar module (E-WOITD) | 1 | ₱184 | ₱184 | LazMall, 93% seller | — |
| 2 | ESP32-CAM + MB programmer board (bundle) | 1 | ₱498 | ₱498 | new listing | — |
| 3 | ESP32 38-pin dev board, CP2102, Type-C (DIYUSER) | 1 | ₱196 | ₱196 | 4.8★ (81) | 822 |
| 4 | KY-008 laser transmitter + laser receiver module pair (Layad Circuits) | 1 pair | ₱253 | ₱253 | 97% seller · 4.7K sold store | — |
| 5 | LM358 dual op-amp DIP-8 (2 pcs, fallback conditioner) | 1 pack | ₱25 | ₱25 | — | 370 |
| 6 | Active buzzer 5V | 1 | ₱30 | ₱30 | 5.0 (42) | 174 |
| 7 | Kingston microSD 16GB Class 10 | 1 | ₱219 | ₱219 | 4.8 (265) | 1.0K |
| 8 | Dupont jumper wires 40-pin (M-F / M-M) | 2 sets | ₱39 | ₱78 | 4.8 (2,814) | 23.2K |
| 9 | Breadboard 830 points (SYB MB-102) | 1 | ₱29 | ₱29 | 4.8 (1,959) | 15.0K |
| 10 | 5V 2A power adapter (DC jack) | 2 | ₱53 | ₱106 | 4.9 (324) | 4.5K |
| 11 | Weatherproof enclosure IP68 ABS (ALLAN 200×100×70) | 1 | ₱220 | ₱220 | 4.9 (1,545) | 14.5K |
| 12 | 2-core 22AWG cable 10 m (far-post laser run) | 1 | ₱51 | ₱51 | — | — |
| 13 | Zip ties 100pcs (mounting) | 1 | ₱12 | ₱12 | — | 153.8K |

**GRAND TOTAL (recommended build): ≈ ₱1,901**

---

## Why the CDM324?

The **CDM324** (InnoSent IPM165 die) is a 24.125 GHz homodyne Doppler radar. Movement in its beam mixes down to an audio-frequency **IF signal whose frequency is directly proportional to speed**:

```
f_doppler (Hz) = 44.7 × speed (km/h)
```

The E-WOITD module sold on Lazada PH is the ICStation-style board with onboard two-stage amplification — the same module family used in public Arduino speed-meter projects (kd8bxp's 24 GHz Doppler speed sketch, JChav02's P10 speed sign). The 38-pin ESP32 just counts pulses on one GPIO; no ADC, no FFT, no radar-gun expertise needed.

**Range:** roughly 10–30 m on a car-sized target (module gain dependent). More than enough for a campus lane.

> **Note on the paper:** the reference study (safeway.pdf) specced **two IR break-beam sensors** for speed and an HC-SR04 as "supplementary detection (optional redundancy)". This build replaces all of them: the **CDM324 radar** takes over speed measurement (direct, physics-exact, no sun-blind receivers, no beam alignment, better multi-lane behavior), and a **KY-008 laser break-beam pair** takes over presence confirmation — same break-beam principle as the paper's IR gates but at visible 650 nm with a photoresistor-based receiver module, so it isn't fooled by sunlight the way raw IR photodiodes are, and it spans the lane on a cheap 2-wire run. The HC-SR04 is dropped entirely (no ultrasonic in this build).

---

## Detailed Listings

### 1. CDM324 24GHz Doppler Radar Sensor Module
- **Price:** ₱177.96–₱184.23 (promo fluctuates) · **Seller rating:** 93% · **Sold:** 22.7K store-wide
- **Seller:** E-WOITD (LazMall) · **Location:** China dropship
- **URL:** https://www.lazada.com.ph/products/pdp-i15517814385.html
- **Backup:** SD&HI same-module listing — https://www.lazada.com.ph/products/pdp-i15590847730.html (₱195.90)
- **Backup 2 (HB100 10.525 GHz alternative, local):** Meterk HB100 Doppler module, ₱207, Bulacan — https://www.lazada.com.ph/products/pdp-i1555917163.html — *different Doppler constant (31.36 Hz per mph); firmware change required if used*
- Note: "Radar Induction Switch" in the name is standard marketing title for this module family — the OUT pin carries the **amplified Doppler IF**, proven speed-readable by multiple public Arduino projects. Verify yours per [HARDWARE.md §IF verification](HARDWARE.md#3-if-signal-verification-do-this-first) before mounting.

### 2. ESP32-CAM + MB Programmer Board (bundle)
- **Price:** ₱498.00 (bundle: AI-Thinker-style ESP32-CAM + MB base board) · **Rating:** new listing
- **URL:** https://www.lazada.com.ph/products/pdp-i15586468265.html
- **Why the MB base board matters:** the classic ESP32-CAM (no USB) needs an external FTDI/CH340 adapter + manual IO0-to-GND jumper dance to flash. The **MB** plugs into the CAM's back and gives it: USB flashing (CH340G), auto-download circuitry, and a 5V power jack — no jumpering. The paper's original BOM had a ₱180 FTDI adapter; the MB replaces it for ₱498 *including the camera board*.
- **Backup (camera board only):** diymore ESP32-CAM-MB programmer board (no camera) ₱357 — https://www.lazada.com.ph/products/pdp-i7757339107.html *(search "ESP32-CAM-MB Download Bottom Board" if link rotates; same seller family)*
- **Backup 2 (parts split):** bare AI-Thinker ESP32-CAM ₱433 (81 sold) + bare MB board ₱111 (JCSYFAC) — works out similar/slightly more expensive than the bundle.

### 3. ESP32 38-Pin Development Board (sensor hub)
- **Price:** ₱196.39 · **Rating:** 4.8★ (81) · **Sold:** 822
- **Seller:** DIYUSER · **Location:** overseas dropship
- **URL:** https://www.lazada.com.ph/products/pdp-i4409112955.html
- **Specs:** ESP-WROOM-32, CP2102 USB-serial, Type-C, dual-core, WiFi/BT — the standard DOIT ESP32 DevKit v1 38-pin layout every tutorial uses.
- **Why 38-pin over 30-pin:** camera-free board = all its pins available; 38-pin is the full DevKit layout (more GPIOs + both I2C/SPI banks broken out) with the most tutorial coverage for sensor work.
- **Backup:** CP2102 WROOM 38Pin Type-C ₱178.80, 293 sold — https://www.lazada.com.ph/products/ (search "CP2102 WROOM ESP32 38Pin Type-C" — NEW listing at 75% off)
- **Backup 2 (local, Bulacan):** PowerMav ESP32 38-pin Type-C ₱233 — search "ESP32 38 Pin Module Type C" on Lazada
- **Why not an ESP32-S3 board:** the S3 variant costs ₱391+ and its camera pins aren't relevant here (no camera on this board); the WROOM-32 is ₱196, runs the same Arduino core, and has more free pins than the project needs.

### 4. KY-008 Laser Transmitter + Laser Receiver Module Pair (presence confirmation)
- **Price:** ₱253.00 (pair — select the "Transmitter&Receiver" variant) · **Seller:** Layad Circuits (Benguet) · **Seller rating:** 97% · 4.7K sold by store · 5-year store, 100% ships in 48 h
- **URL:** https://www.lazada.com.ph/products/pdp-i5141273577.html
- **Backup (TX only):** KY-008 module from Fulabs ₱44 — https://www.lazada.com.ph/products/pdp-i5367051449.html (pair it with a separate laser-receiver module; e.g. "Laser Receiver Sensor" ₱165, Laguna — https://www.lazada.com.ph/products/pdp-i4352848577.html)
- **Backup 2 (kit):** "KY-008 Laser Transmitter + Non-Modulator Laser Receiver Module Kit" ₱626 — https://www.lazada.com.ph/products/pdp-i15568605059.html
- Role: **KY-008 TX** sits on the far post (650 nm red laser, ~5 mW, always-on); the **receiver module** (photodiode/photoresistor front-end, comparator with digital DO output) sits on the hub pole. A vehicle crossing the lane breaks the beam → DO changes state → hub flags "vehicle physically present" to confirm the radar event. This replaces the paper's IR break-beam concept with a visible-light version that resists sunlight interference.
- KY-008 pinout: **S = signal** (tie to 5 V for always-on — simplest), **middle = +5 V**, **− = GND**. Draws < 30 mA.
- **Safety note:** 5 mW / 650 nm is Class 3R-adjacent — never look into the beam, don't aim at eye level of drivers/pedestrians; mount it low (plate height) and aim it across the lane at the receiver, not along it.

### 5. LM358 Dual Op-Amp DIP-8 (2 pcs) — signal conditioning fallback
- **Price:** ₱25.00 (2 pieces) · **Sold:** 370 · **Location:** Bulacan
- **URL:** https://www.lazada.com.ph/products/pdp-i3251664504.html *(same seller family as prior verified listing — if rotated: search "LM358 DIP" pick Bulacan seller)*
- Alternative prebuilt: "LM358 100× Gain Signal Amplification Module" ₱148, Bulacan — zero soldering if your module's IF output turns out weak.
- Note: Only needed if your CDM324 variant's output is too weak or already-digitized to pulse-count ([HARDWARE.md §4](HARDWARE.md#4-fallback-lm358-signal-conditioner)). Most modules don't need it.

### 6. Active Buzzer 5V (alarm/alert)
- **Price:** ₱30.00 · **Rating:** 5.0 (42) · **Sold:** 174 · **Location:** Metro Manila
- **URL:** https://www.lazada.com.ph/products/pdp-i4216290365.html
- Note: **Active** buzzer (built-in oscillator). "1-PC per ORDER". Select 5V.
- **Backup:** ₱23, 73 sold, (8), Benguet — https://www.lazada.com.ph/products/pdp-i4908420831.html

### 7. Kingston microSD Card 16GB Class 10 (U1)
- **Price:** ₱219 (select **16GB** variant — page defaults to 512GB at ₱549) · **Rating:** 4.8 (265) · **Sold:** 1.0K
- **Seller:** vgsvny store · **Location:** Quezon City
- **URL:** https://www.lazada.com.ph/products/pdp-i1806785852.html
- Note: Slides into the **CAM board's TF slot** — local backup of every violation photo ([FIRMWARE.md](FIRMWARE.md) camera sketch saves alongside WiFi upload). Class 10 / U1 minimum.
- **Backup:** https://www.lazada.com.ph/products/pdp-i15448124514.html

### 8. Dupont Jumper Wires 40-pin (Male-Female + Male-Male)
- **Price:** ₱39.00 per set (2 sets budgeted: M-F module→breadboard, M-M rail jumps)
- **Sold:** 23.2K · **Rating:** 4.8 (2,814) · **Location:** Bulacan
- **URL:** https://www.lazada.com.ph/products/pdp-i5992263.html

### 9. Breadboard 830 Tie Points (SYB MB-102, full-size)
- **Price:** ₱29.00 · **Sold:** 15.0K · **Rating:** 4.8 (1,959) · **Location:** Bulacan
- **URL:** https://www.lazada.com.ph/products/pdp-i8845741.html

### 10. 5V 2A Power Supply Adapter (DC 5.5×2.5mm jack)
- **Price:** ₱53.00 each, **2 pcs budgeted** · **Sold:** 4.5K · **Rating:** 4.9 (324) · **Location:** Bulacan
- **URL:** https://www.lazada.com.ph/products/pdp-i3062233761.html
- Note: one adapter powers the 38-pin hub (radar + beam receiver + buzzer **and the far-post KY-008 TX over the 22AWG run** — it draws < 30 mA, negligible), one powers the CAM-MB board. WiFi bursts + camera draw peaks; separate adapters mean a camera reboot can't brown-out the radar mid-measurement.
- **Backup:** ₱55, 5.5K sold — https://www.lazada.com.ph/products/pdp-i2442570467.html

### 11. Weatherproof Enclosure — ALLAN IP68 ABS Junction Box
- **Price:** ₱220 (200×100×70mm variant) · **Rating:** 4.9 (1,545) · **Sold:** 14.5K
- **Seller:** ALLAN.PH (LazMall) · **Location:** Quezon City
- **URL:** https://www.lazada.com.ph/products/pdp-i2890018648.html
- Note: 200×100×70 fits the 830-point breadboard + 38-pin board + CAM-MB + radar wiring. **24 GHz passes through ABS plastic** — radar can sit behind a plain plastic wall (no metal!). The CAM needs a window — see [HARDWARE.md §7](HARDWARE.md#7-enclosure-assembly-ip68-abs-200x100x70-mm).
- **Budget alt:** IP65 ABS box ₱27.50 — https://www.lazada.com.ph/products/pdp-i5215693124.html (light rain only)

### 12. 2-Core 22AWG Cable 10 m (far-post laser run)
- **Price:** ₱50.51 (10 m roll, red/black) · **Location:** Bulacan (local express)
- **URL:** https://www.lazada.com.ph/products/pdp-i5407853518.html
- Role: powers the far-post KY-008 transmitter (5 V + GND) from the hub enclosure. 22AWG copper at < 30 mA has millivolts of drop even at 10 m — the laser runs at full brightness. Run it along the ground/curb (UV-rated ties, buried conduit sleeve if possible) — it is the system's only cable.
- **Backup:** JCSYFAC same-spec ₱59.38 — https://www.lazada.com.ph/products/pdp-i15496940788.html

### 13. Cable/Zip Ties 100pcs (mounting + cable management)
- **Price:** ₱12.00 · **Sold:** 40K+ · **Location:** Bulacan
- **URL:** https://www.lazada.com.ph/products/pdp-i7109771.html *(or search "zip ties 100pcs")*

---

## Budget variants

| Build | What changes | Total |
|---|---|---:|
| **Recommended** (as above) | — | **≈ ₱1,901** |
| **Bare-bones bench POC** | IP65 box (−₱192); one adapter powering both boards + rail cap (−₱53); skip zip ties (−₱12); one jumper set (−₱39) | ≈ ₱1,605 |
| **Deluxe** | Prebuilt LM358 amp module instead of ICs (+₱123); small IP65 box (e.g. ₱60) on the far post for the laser TX (+₱60) | ≈ ₱2,084 |
| **No far post available?** | Skip the cross-lane beam (−₱253 −₱51) and run radar-only with `MIN_SPEED_KPH` raised — records stay but lose the two-sensor confirm flag | ≈ ₱1,597 |

Parts verified Sep 6–8, 2026. Re-check prices before ordering — Lazada promo pricing moves daily.
