# Block Diagram

Hardware blocks and the signals that connect them — the physical view of SafeWay.

---

## 1. Overall System Block Diagram

```
 FAR POST                    HUB POLE (weatherproof enclosure)                 LANE
┌──────────────┐   650 nm    ┌────────────────────────┐
│  KY-008      │  laser dot  │  LASER RECEIVER        │
│  laser TX    │────────────►│  module (comparator)   │──DO──► ┐
└──────┬───────┘  across the │  behind clear window    │        │
       │               lane   └────────────────────────┘        ▼
       │ 22AWG pair                                        ┌─────────────┐
       │ (5V + GND)                                        │   ESP32     │
┌──────┴──────────────────────────────────────────────┐   │  38-pin HUB │
│  ADAPTER #1 (5V 2A)                                 │   │             │
│  ├─► 38-pin hub 5V rail                              │◄──┤ GPIO 25     │
│  ├─► CDM324 radar VCC        ┌──────────────────┐   │   │ (beam ISR,  │
│  ├─► laser receiver VCC      │  CDM324 24 GHz    │   │   │  50 ms de-  │
│  └─► 22AWG pair ─► far TX    │  Doppler radar    │───┼──►│  bounce)     │
└──────────────────────────────│  (speed sensing)  │OUT│   │             │
                               └──────────────────┘   │   │ GPIO 34     │
                                                     │   │ (pulse ISR,  │
                               ┌──────────────────┐  │   │  RISING)     │
                               │  ACTIVE BUZZER   │◄─┼───┤ GPIO 27     │
                               │  (5V, overspeed) │  │   │             │
                               └──────────────────┘  │   │ event logic │
                                                     │   │ Hz→km/h     │
                                                     │   │ WiFi client │
                                                     │   └──────┬──────┘
                                                     │          │ ① WiFi —
                                                     │          │ GET /capture
  ADAPTER #2 (5V 2A)                                │          ▼
  └─► ESP32-CAM-MB 5V   ┌────────────────────────────┴──────────────────┐
                        │ ESP32-CAM (OV2640) on MB programmer            │
                        │  • /capture  → JPEG  (+ auto-save to microSD) │
                        │  • /stream   → live MJPEG                      │
                        │  • microSD 16 GB Class 10 (photo failover)     │
                        └───────────────┬────────────────────────┬───────┘
                                        │                        │ ③ <img> tag
                              ② WiFi —  │ JSON POST              │ pulls MJPEG
                              violation │ over the incident      │ directly
                              event     ▼                        ▼
                        ┌───────────────────────────┐   ┌──────────────────────┐
                        │ SERVER (one process)      │   │ SSU BROWSER          │
                        │ FastAPI + uvicorn :8000   │◄──│ Monitoring Dashboard │
                        │  • SQLite safeway.db      │ ④ │ • violation table    │
                        │  • uploads/ photo store   │GET│ • photo pane         │
                        │  • plate recognition      │   │ • live lane feed     │
                        │   (OpenALPR / Tesseract)  │   │ • review actions     │
                        └───────────────────────────┘   └──────────────────────┘
```

**Flows:** ① hub pulls the evidence photo from the CAM · ② hub uploads the incident JSON · ③ dashboard's live feed goes browser→CAM directly (no server relay) · ④ dashboard reads the API.

---

## 2. Power Distribution

```
 ADAPTER #1 (5 V 2A) ──┬── 38-pin hub VIN (board + WiFi bursts)
                      ├── CDM324 VCC          (~30–60 mA)
                      ├── laser receiver VCC  (~10 mA)
                      └── 22AWG 2-core run (10 m) ──► far-post KY-008 TX (<30 mA)

 ADAPTER #2 (5 V 2A) ──── ESP32-CAM-MB 5V (board + OV2640 + WiFi + SD)
```

- Separate adapters ⇒ a camera reboot can never brown-out the radar mid-measurement.
- One rail cap recommended when bench-testing from a single supply.
- Far TX over 22AWG: <30 mA draw — negligible drop over 10 m, no far-side outlet needed.

---

## 3. Signal Chain Summary

| Signal | Source → Destination | Conditioning | Meaning |
|---|---|---|---|
| Doppler IF pulses | CDM324 OUT → GPIO 34 | divider **if** §3 scope check reads >3.3 V | pulse rate = 44.7 Hz per km/h |
| Beam level | laser RX DO → GPIO 25 | internal pull-up; divider only if DO swings to 5 V | LOW/HIGH = beam intact/broken (`BEAM_BREAKS_LOW`) |
| Beam (optical) | KY-008 TX → receiver window | 650 nm dot across the lane | blocked = solid object crossing |
| Alert | GPIO 27 → buzzer | direct (active 5V module) | HIGH while kph > limit |
| Evidence | CAM `/capture` → hub (WiFi) | base64 in memory | JPEG + microSD save |
| Incident | hub → API (WiFi) | JSON POST | speed, limit, doppler_hz, confirmed, photo |

---

Electrical details: [HARDWARE.md](HARDWARE.md) · pin map & wiring tables
Data flow logic: [FLOWCHART.md](FLOWCHART.md) · Layers & interfaces: [SYSTEM-ARCHITECTURE.md](SYSTEM-ARCHITECTURE.md)
