# SafeWay — Bill of Materials (BOM)

**Project:** IoT Vehicle Speed Monitoring System (ESP32-CAM + KY-008 laser break-beam)
**Source:** Lazada Philippines · Prices checked **Sep 6, 2026**
**Criteria:** ⭐3+ rating, 8+ ratings count, proven sales (sold count), local PH sellers preferred, cheapest that qualifies.
**Note:** Lazada prices move with vouchers/promos — treat totals as estimates (±10%).

---

## Summary Table

| # | Component | Qty | Unit Price | Subtotal | Rating | Sold |
|---|-----------|-----|-----------:|---------:|--------|------|
| 1 | KY-008 650nm Laser Transmitter | 2 | ₱40 | ₱80 | 5.0★ (42) | 294 |
| 2 | Laser Receiver Module (non-modulator tube) | 2 | ₱113 | ₱226 | — *see note | — * |
| 3 | ESP32-CAM (AI-Thinker, OV2640 2MP) | 1 | ₱549 | ₱549 | 5.0★ (9) | 41 |
| 4 | CH340 USB-to-TTL adapter (programming) | 1 | ₱55 | ₱55 | 5.0★ (69) | 393 |
| 5 | Kingston microSD 16GB Class 10 | 1 | ₱219 | ₱219 | 4.8★ (265) | 1.0K |
| 6 | Active buzzer 5V | 1 | ₱30 | ₱30 | 5.0★ (42) | 174 |
| 7 | HC-SR04 ultrasonic sensor (supplementary) | 1 | ₱35 | ₱35 | 4.8★ (64) | 790 |
| 8 | Dupont jumper wires 40-pin (M-F / M-M / F-F) | 1 set | ₱39 | ₱39 | — * | 23.2K |
| 9 | Breadboard 830 points (SYB MB-102) | 1 | ₱29 | ₱29 | — * | 15.0K |
| 10 | AMS1117 3.3V regulator module (2pcs) | 1 | ₱110 | ₱110 | — * | 115 |
| 11 | 5V 2A power adapter (DC jack) | 1 | ₱53 | ₱53 | — * | 4.5K |
| 12 | Weatherproof enclosure IP68 ABS (ALLAN) | 1 | ₱220–300 | ₱220 | 4.9★ (1,545) | 14.5K |
| 13 | Zip ties 100pcs (mounting) | 1 | ₱12 | ₱12 | — * | 153.8K |

**GRAND TOTAL (recommended build): ≈ ₱1,657**

> \* Listings where the search card shows sold counts but star value wasn't captured live — sold counts and rating counts are shown; ratings count ≥ 8 everywhere except the New Arrival receiver (see note below).
>
> **Laser pair qty note:** Speed = distance ÷ time, so two break-beam gates at a known separation give a clean speed reading on one ESP32-CAM. Start with 1 pair (₱153) if prototyping a single gate first — **budget build total: ≈ ₱1,226.50** (1 laser pair + SMD AMS1117 ₱25 + IP65 box ₱27.50 instead of module/IP68).

---

## Detailed Listings

### 1. KY-008 650nm Laser Transmitter Module
- **Price:** ₱40.00 · **Rating:** 5.0★ (42 ratings) · **Sold:** 294
- **Seller:** Makerlab PH (98% positive, Top Seller) · **Location:** Bulacan
- **URL:** https://www.lazada.com.ph/products/pdp-i3541932131.html
- Note: 3-pin module (S, VCC, GND) — the standard break-beam transmitter.

### 2. Laser Receiver Module (non-modulator tube, pairs with KY-008)
- **Price:** ₱113.00 (receiver variant) · **Seller:** Layad Circuits (97% seller rating, 4.7K store sold, 5-yr store) · **Location:** Benguet
- **URL (same listing, variants):** https://www.lazada.com.ph/products/pdp-i5141273577.html
  - Transmitter + Receiver **pair:** ₱253 · **Receiver only:** ₱113 (select the "Laser Receiver" variant)
- ⚠️ Listing is **New Arrival** (no product ratings yet) — qualified via seller track record. Same seller sells the KY-008-compatible tube receiver.
- **Alternative (rated kit):** KY-008 + receiver tube kit listings exist around ₱626 from Metro Manila sellers on the same search — use if you want rated listings only. Cheapest verified combo remains Makerlab KY-008 (₱40) + this receiver (₱113) = **₱153/pair**.

### 3. ESP32-CAM (AI-Thinker type, OV2640 2MP camera)
- **Price:** ₱549.00 · **Rating:** 5.0★ (9 ratings) · **Sold:** 41
- **Seller:** Makerlab PH (98% seller rating, 10-yr store) · **Location:** Bulacan
- **URL:** https://www.lazada.com.ph/products/pdp-i4935947534.html
- Note: "ESP32 CAM 2MP Starter with OV2640" — get the **with-camera** variant (some listings sell board-only).
- **Backup:** https://www.lazada.com.ph/products/pdp-i4054216963.html (ESP32-CAM WiFi+BT w/ OV2640)
- ⚠️ Avoid listings whose base price (~₱141) is for the "FT232 burner" variant only — misleading. Genuine ESP32-CAM boards run ₱500+ on Lazada PH.

### 4. CH340 USB-to-TTL Serial Adapter (for flashing the ESP32-CAM)
- **Price:** ₱55.00 · **Rating:** 5.0★ (69 ratings) · **Sold:** 393
- **Seller:** Makerlab PH · **Location:** Bulacan
- **URL:** https://www.lazada.com.ph/products/pdp-i2500959987.html
- Note: ESP32-CAM has no onboard USB — this adapter is required for programming. Set 5V when flashing (board has onboard regulator), 3.3V for direct GPIO work.
- **Backup:** CP2102 USB-UART (₱129, 30 sold, 9 ratings, Valenzuela) — https://www.lazada.com.ph/products/pdp-i4888499071.html

### 5. Kingston microSD Card 16GB Class 10 (U1)
- **Price:** ₱219 (select **16GB** variant — page defaults to 512GB at ₱549) · **Rating:** 4.8★ (265) · **Sold:** 1.0K
- **Seller:** vgsvny store (91% seller rating, Top 12 Seller for Memory Cards, 5-yr store, 93.2K store sold) · **Location:** Quezon City
- **URL:** https://www.lazada.com.ph/products/pdp-i1806785852.html
- Note: ESP32-CAM photo capture saves to microSD. Class 10 / U1 is the minimum for reliable capture. COD available.
- **Backup:** https://www.lazada.com.ph/products/pdp-i15448124514.html

### 6. Active Buzzer 5V (alarm/alert)
- **Price:** ₱30.00 · **Rating:** 5.0★ (42 ratings) · **Sold:** 174
- **Seller:** PC.Po (96% positive) · **Location:** Metro Manila
- **URL:** https://www.lazada.com.ph/products/pdp-i4216290365.html
- Note: **Active** buzzer (built-in oscillator — DC in, sound out, no tone library needed). "1-PC per ORDER". Select 5V.
- **Backup:** ₱23, 73 sold, (8), Benguet — https://www.lazada.com.ph/products/pdp-i4908420831.html

### 7. HC-SR04 Ultrasonic Sensor (supplementary vehicle detection)
- **Price:** ₱35.00 · **Rating:** 4.8★ (64 ratings) · **Sold:** 790
- **Seller:** Circuitrocks (97% seller rating, 253.4K store sold) · **Location:** Bulacan
- **URL:** https://www.lazada.com.ph/products/pdp-i235234290.html
- **Backup:** ₱45, 7.7K sold, (989), Bulacan — https://www.lazada.com.ph/products/pdp-i123573087.html

### 8. Dupont Jumper Wires 40-pin (Male-Female / Male-Male / Female-Female)
- **Price:** ₱39.00 (per 40-pin cable; select type & length variants: 10/20/30cm) · **Sold:** 23.2K · (2,814 ratings)
- **Seller:** Bulacan-based electronics store · **Location:** Bulacan
- **URL:** https://www.lazada.com.ph/products/pdp-i5992263.html
- Note: You'll want **at least 2 types** (M-F for modules→breadboard, M-M for breadboard rails). Budget ~₱78 for two sets.
- **Backup:** ₱25, 2.6K sold, (424), Valenzuela

### 9. Breadboard 830 Tie Points (SYB MB-102, full-size)
- **Price:** ₱29.00 · **Sold:** 15.0K · (1,959 ratings)
- **Location:** Bulacan
- **URL:** https://www.lazada.com.ph/products/pdp-i8845741.html
- Note: Standard 830-point solderless breadboard; fits ESP32-CAM breakout wiring + regulator module.

### 10. AMS1117 3.3V Regulator — **Module version (recommended, 2pcs)**
- **Price:** ₱110.00 (2 pieces) · **Sold:** 115 · (16 ratings)
- **Location:** Bulacan
- **URL:** https://www.lazada.com.ph/products/pdp-i115879545.html
- Note: Breadboard-friendly breakout with input/output caps — drop 5V→3.3V for clean ESP32-CAM/laser sensor supply.
- **Budget alt (needs soldering/adapter):** 2pcs bare SOT-223 AMS1117-3.3 — ₱25, 240 sold, (50), Bulacan — https://www.lazada.com.ph/products/pdp-i3251664504.html

### 11. 5V 2A Power Supply Adapter (DC 5.5×2.5mm jack)
- **Price:** ₱53.00 · **Sold:** 4.5K · (324 ratings)
- **Location:** Bulacan
- **URL:** https://www.lazada.com.ph/products/pdp-i3062233761.html
- Note: 2A headroom is right for ESP32-CAM + camera + WiFi bursts (brownouts happen below 2A). Pair with a DC jack breakout or cut/splice to breadboard rails.
- **Backup:** ₱55, 5.4K sold, (627), Rizal — https://www.lazada.com.ph/products/pdp-i2442570467.html

### 12. Weatherproof Enclosure — ALLAN IP68 ABS Junction Box
- **Price:** ₱220–300 (size variant; ~₱220 for 200×100×70mm) · **Rating:** 4.9★ (1,545) · **Sold:** 14.5K
- **Seller:** ALLAN.PH (LazMall, 98% seller rating, 1M store sold, Top 14 Seller Network Components) · **Location:** Quezon City
- **URL:** https://www.lazada.com.ph/products/pdp-i2890018648.html
- Sizes available: 50×50 / 100×100×70 / 150×150×70 / **200×100×70** ✅ / 250×200×80 / 250×200×120 mm — the 200×100×70 fits the 830-point breadboard (165×55mm) + ESP32-CAM + wiring.
- **Budget alt:** IP65 ABS junction box ₱27.50, 4.2K sold, (147), Quezon City — https://www.lazada.com.ph/products/pdp-i5215693124.html (light rain protection only; IP68 is the safe choice for pole-mounted outdoor use)

### 13. Cable/Zip Ties 100pcs (mounting + cable management)
- **Price:** ₱12.00 · **Sold:** 153.8K · (20,507 ratings)
- **Location:** Cavite
- **URL:** https://www.lazada.com.ph/products/pdp-i4267223500.html
- Note: UV-resistant nylon — for strapping enclosure to pole/bracket and securing sensor wiring. Screws/brackets: repurpose or add a generic wall-plug screw set (~₱30–50).
- **Backup:** ₱25, 53.5K sold, (7,063), Pasig

---

## Cost Summary

| Build | Total |
|-------|------:|
| **Recommended** (2 laser gates, AMS1117 module, IP68 box) | **≈ ₱1,657** |
| **Budget** (1 laser gate, SMD regulator, IP65 box) | **≈ ₱1,226.50** |
| + Shipping (est., most sellers Bulacan/QC/Cavite) | + ₱30–60 per seller shipment |

**Seller clustering tip:** Makerlab PH (Bulacan) supplies items 1, 3, 4 + likely 8–11 neighbors — one-store batching cuts shipping and usually gets free-shipping vouchers.

## Not included (assumed on hand / out of scope)
- Soldering iron + solder (if using module versions, mostly avoidable)
- Dupont crimps / heat shrink, USB extension for flashing
- Server/cloud costs (dashboard API hosting)
- Mounting pole/bracket hardware (site-dependent)
