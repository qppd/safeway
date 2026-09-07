# Block Diagram

Hardware blocks and the signals that connect them — the physical view of SafeWay. Diagrams are **Mermaid** and render natively on GitHub.

---

## 1. Overall System Block Diagram

```mermaid
flowchart TB
    subgraph FAR["FAR POST"]
        TX["KY-008 laser TX"]
    end

    subgraph POLE["HUB POLE — weatherproof enclosure"]
        RX["Laser receiver module<br/>comparator behind clear window"]
        RADAR["CDM324 24 GHz<br/>Doppler radar"]
        BUZZ["Active buzzer 5 V"]
        HUB["ESP32 38-pin HUB<br/>pulse ISR — GPIO 34<br/>beam ISR + 50 ms debounce — GPIO 25<br/>buzzer — GPIO 27<br/>event logic · Hz→km/h · WiFi"]
        CAM["ESP32-CAM (OV2640) on MB board<br/>/capture JPEG + microSD save<br/>/stream live MJPEG<br/>microSD 16 GB failover"]
        P1["Adapter #1 · 5 V 2 A"]
        P2["Adapter #2 · 5 V 2 A"]
    end

    subgraph SRV["SERVER — one process"]
        API["FastAPI + uvicorn :8000<br/>SQLite safeway.db · uploads/<br/>plate recognition (OpenALPR / Tesseract)"]
    end

    subgraph SSU["SSU BROWSER"]
        DASH["Monitoring dashboard<br/>violation table · photo pane · live feed"]
    end

    TX -- "650 nm dot across the lane" --> RX
    RX -- "DO → GPIO 25" --> HUB
    RADAR -- "OUT → GPIO 34" --> HUB
    HUB -- "GPIO 27" --> BUZZ
    HUB -- "① GET /capture — fetch photo" --> CAM
    HUB -- "② POST /api/incidents — JSON" --> API
    CAM -. "③ MJPEG direct — img tag" .-> DASH
    API -- "④ GET incidents + photos" --> DASH
    P1 -.-> HUB
    P1 -.-> RADAR
    P1 -.-> RX
    P1 -- "22AWG 2-core run (10 m)" --> TX
    P2 -.-> CAM
```

**Legend:** solid arrows = data · dotted arrows = power · numbered flows:

- ① hub pulls the evidence photo from the CAM (WiFi)
- ② hub uploads the incident JSON (WiFi)
- ③ dashboard live feed goes browser→CAM directly (LAN) — no server relay
- ④ dashboard reads the API

---

## 2. Power Distribution

```mermaid
flowchart LR
    A1["Adapter #1 · 5 V 2 A"] --> HUBV["38-pin hub VIN<br/>board + WiFi bursts"]
    A1 --> RRV["CDM324 VCC<br/>~30–60 mA"]
    A1 --> LRV["Laser receiver VCC<br/>~10 mA"]
    A1 --> WIRE["22AWG 2-core run<br/>10 m"] --> TXV["Far-post KY-008 TX<br/>under 30 mA"]
    A2["Adapter #2 · 5 V 2 A"] --> CAMV["ESP32-CAM-MB 5 V<br/>board + OV2640 + WiFi + SD"]
```

- Separate adapters ⇒ a camera reboot can never brown-out the radar mid-measurement.
- One rail cap recommended when bench-testing from a single supply.
- Far TX over 22AWG: under 30 mA draw — negligible drop over 10 m, no far-side outlet needed.

---

## 3. Signal Chain Summary

| Signal | Source → Destination | Conditioning | Meaning |
|---|---|---|---|
| Doppler IF pulses | CDM324 OUT → GPIO 34 | divider **if** §3 scope check reads >3.3 V | pulse rate = 44.7 Hz per km/h |
| Beam level | laser RX DO → GPIO 25 | internal pull-up; divider only if DO swings to 5 V | LOW/HIGH = beam intact/broken (`BEAM_BREAKS_LOW`) |
| Beam (optical) | KY-008 TX → receiver window | 650 nm dot across the lane | blocked = solid object crossing |
| Alert | GPIO 27 → buzzer | direct (active 5V module) | HIGH while kph over limit |
| Evidence | CAM `/capture` → hub (WiFi) | base64 in memory | JPEG + microSD save |
| Incident | hub → API (WiFi) | JSON POST | speed, limit, doppler_hz, confirmed, photo |

---

Electrical details: [HARDWARE.md](HARDWARE.md) · pin map & wiring tables
Data flow logic: [FLOWCHART.md](FLOWCHART.md) · Layers & interfaces: [SYSTEM-ARCHITECTURE.md](SYSTEM-ARCHITECTURE.md)
