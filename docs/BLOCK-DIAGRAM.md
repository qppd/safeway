# Block Diagram

Hardware blocks and the signals that connect them — the physical view of SafeWay. Diagrams are **Mermaid** and render natively on GitHub.

---

## 1. Overall System Block Diagram

```mermaid
flowchart TB
    subgraph FAR["FAR POST — self-powered"]
        TX1["KY-008 laser TX #1<br/>supply #1"]
        TX2["KY-008 laser TX #2<br/>supply #2"]
    end

    subgraph POLE["HUB POLE — weatherproof enclosure"]
        RX1["Laser receiver #1<br/>comparator behind clear window"]
        RX2["Laser receiver #2<br/>behind its own window"]
        RADAR["CDM324 24 GHz<br/>Doppler radar"]
        BUZZ["Active buzzer 5 V"]
        HUB["ESP32 38-pin HUB<br/>pulse ISR — GPIO 34<br/>beam #1 ISR + 50 ms debounce — GPIO 25<br/>buzzer — GPIO 27<br/>event logic · Hz→km/h · WiFi"]
        CAM["ESP32-S3 WROOM CAM (OV5640)<br/>beam #2 ISR — GPIO 21 → self-triggered snapshot<br/>cached plate frame on /capture<br/>/stream polled live frame<br/>microSD 16 GB failover"]
        P3["Supply #3 · 5 V 2 A"]
        P4["Supply #4 · 5 V 2 A"]
    end

    subgraph SRV["SERVER — one process"]
        API["FastAPI + uvicorn :8000<br/>SQLite safeway.db · uploads/<br/>plate recognition (OpenCV + Tesseract)"]
    end

    subgraph SSU["SSU BROWSER"]
        DASH["Monitoring dashboard<br/>violation table · photo pane · live feed"]
    end

    TX1 -- "650 nm dot #1 across the lane" --> RX1
    TX2 -- "650 nm dot #2 across the lane (offset)" --> RX2
    RX1 -- "DO → GPIO 25" --> HUB
    RX2 -- "DO → GPIO 21" --> CAM
    RADAR -- "OUT → GPIO 34" --> HUB
    HUB -- "GPIO 27" --> BUZZ
    HUB -- "① GET /capture at beam-break — receives CAM's cached beam-moment frame" --> CAM
    HUB -- "② POST /api/incidents — JSON" --> API
    CAM -. "③ polled frame — img tag, ~1/s" .-> DASH
    API -- "④ GET incidents + photos" --> DASH
    P3 -.-> HUB
    P3 -.-> RADAR
    P3 -.-> RX1
    P4 -.-> CAM
    P4 -.-> RX2
```

**Legend:** solid arrows = data · dotted arrows = power · numbered flows:

- ① at beam-break, the hub pulls the evidence photo from the CAM (WiFi) — what it receives is the **beam-moment frame the CAM already captured itself** (vehicle at the pole, plate in frame)
- ② hub uploads the incident JSON (WiFi)
- ③ dashboard live feed goes browser→CAM directly (LAN) — no server relay
- ④ dashboard reads the API
- **No cable crosses the road** — TX #1 and TX #2 each run on their own far-post supply; the two poles talk over WiFi only

---

## 2. Power Distribution — four independent supplies

```mermaid
flowchart LR
    S1["Supply #1 · far post<br/>compact 5 V ≥1 A"] -.-> TX1V["KY-008 TX #1<br/>under 30 mA"]
    S2["Supply #2 · far post<br/>compact 5 V ≥1 A"] -.-> TX2V["KY-008 TX #2<br/>under 30 mA"]
    S3["Supply #3 · hub pole<br/>5 V 2 A"] -.-> HUBV["38-pin hub VIN<br/>board + WiFi bursts"]
    S3 -.-> RRV["CDM324 VCC<br/>~30–60 mA"]
    S3 -.-> LRV["Laser receiver #1 VCC<br/>~10 mA"]
    S4["Supply #4 · hub pole<br/>5 V 2 A"] -.-> CAMV["ESP32-S3 WROOM CAM 5 V<br/>board + OV5640 + WiFi + SD"]
    S4 -.-> LRV2["Laser receiver #2 VCC<br/>~10 mA"]
```

- Four supplies, zero shared rails ⇒ a camera reboot can never brown-out the radar mid-measurement; a dead far-post supply can never dim the other beam; a hub buzzer/WiFi burst can never ripple the CAM's rail.
- **No cable crosses the road** — each far-post TX is self-powered (compact USB charger inside its housing).
- One rail cap recommended on each board's supply when bench-testing from a single source.

---

## 3. Signal Chain Summary

| Signal | Source → Destination | Conditioning | Meaning |
|---|---|---|---|
| Doppler IF pulses | CDM324 OUT → GPIO 34 | divider **if** §3 scope check reads >3.3 V | pulse rate = 44.7 Hz per km/h |
| Beam #1 level | laser RX #1 DO → GPIO 25 | internal pull-up; divider only if DO swings to 5 V | LOW/HIGH = beam intact/broken (`BEAM_BREAKS_LOW`) |
| Beam #2 level | laser RX #2 DO → GPIO 21 (CAM) | `INPUT_PULLUP`; divider **mandatory** if DO >3.3 V (S3 not 5 V tolerant) | drives the CAM's self-triggered snapshot (`CAM_BEAM_BREAKS_LOW`) |
| Beam #1/#2 (optical) | KY-008 TX #1/#2 → receiver windows | 650 nm dots across the lane, offset a few cm | blocked = solid object crossing |
| Alert | GPIO 27 → buzzer | direct (active 5V module) | HIGH while kph over limit |
| Evidence | CAM self-trigger + `/capture` → hub (WiFi) | beam #2 break → cached PSRAM frame; hub fetches on beam #1 break | plate-frame JPEG (vehicle at the pole) + microSD save |
| Incident | hub → API (WiFi) | JSON POST | speed, limit, doppler_hz, confirmed, photo |

---

Electrical details: [HARDWARE.md](HARDWARE.md) · pin map & wiring tables · full schematic: [wiring/circuit_image.png](../wiring/circuit_image.png)
Data flow logic: [FLOWCHART.md](FLOWCHART.md) · Layers & interfaces: [SYSTEM-ARCHITECTURE.md](SYSTEM-ARCHITECTURE.md)
