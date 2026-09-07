# System Architecture

Layers, boundaries, and interfaces — how SafeWay is organized as a system, from photons to dashboard.

---

## 1. Layered View

```
┌─────────────────────────────────────────────────────────────────┐
│ L5  PRESENTATION      SSU browser: dashboard.html (Tailwind +    │
│                      vanilla JS) — table, photo pane, live feed  │
├─────────────────────────────────────────────────────────────────┤
│ L4  APPLICATION       FastAPI :8000 — POST/GET/PATCH incidents, │
│                      photo serve, cam-status, dashboard hosting │
├─────────────────────────────────────────────────────────────────┤
│ L3  DATA & INTELLIGENCE   SQLite (incidents) · uploads/ store ·  │
│                      plate recognition (OpenALPR / Tesseract)    │
├─────────────────────────────────────────────────────────────────┤
│ L2  EDGE DEVICES      ESP32 hub (sensing + event logic +        │
│                      reporting) · ESP32-CAM (evidence + stream)  │
├─────────────────────────────────────────────────────────────────┤
│ L1  SENSING & PHYSICS CDM324 Doppler radar · KY-008 + receiver  │
│                      break-beam · OV2640 imager · buzzer        │
└─────────────────────────────────────────────────────────────────┘
        L1→L2 electrical (GPIO)   L2→L4 WiFi HTTP   L4→L5 HTTP/HTML
```

**Key boundary decisions:**

- **Sensing on the edge, intelligence on the server.** The hub does physics (Hz→km/h), thresholding, and event assembly — everything that must happen in real time. The server does OCR and storage — everything that can wait. A 300 ms loop on the hub never competes with plate recognition on the server.
- **Two boards, one job each.** Camera + SD consumes an ESP32-CAM's pins and heap; splitting sensors from imaging means a camera crash can't take down speed measurement (and vice versa). L2 is deliberately two independent fault domains.
- **Evidence has two paths.** Photo goes hub→server immediately when WiFi is up; microSD on the CAM is the durable copy either way. The system degrades gracefully, never loses evidence.

## 2. Component Responsibilities

| Component | Owns | Never does |
|---|---|---|
| CDM324 radar | IF pulse generation (44.7 Hz/km/h) | — |
| Break-beam pair | presence edge across the lane | speed (radar's job) |
| ESP32 hub | pulse counting, beam ISR+debounce, Hz→km/h, cosine correction, peak-hold, buzzer, photo fetch, incident POST | OCR, long-term storage |
| ESP32-CAM | capture on demand, MJPEG stream, microSD save | speed logic, upload |
| FastAPI server | validation, server-side timestamps, photo storage, OCR orchestration, dashboard hosting | sensing decisions |
| Dashboard | read, filter, review, view live feed | write raw records (only PATCH review) |

## 3. Interfaces (the contract set)

| # | Interface | Protocol | Contract |
|---|---|---|---|
| I1 | Radar OUT → hub GPIO 34 | electrical, RISING ISR | pulses; rate ∝ speed |
| I2 | Beam RX DO → hub GPIO 25 | electrical, CHANGE ISR | level encodes intact/broken (`BEAM_BREAKS_LOW`) |
| I3 | Buzzer ← GPIO 27 | electrical | HIGH = over limit |
| I4 | Hub → CAM `GET /capture` | WiFi HTTP | returns JPEG (+ SD save) |
| I5 | Hub → CAM `GET /stream` | WiFi HTTP | MJPEG (dashboard pulls it too, I7) |
| I6 | Hub → API `POST /api/incidents` | WiFi HTTP JSON | device_id, speed_kph, limit_kph, doppler_hz, confirmed, photo_b64 → 201 |
| I7 | Browser → CAM `<img>` | LAN HTTP | MJPEG, direct — no server relay |
| I8 | Browser → API GET/PATCH | HTTP JSON | list/filter incidents; mark reviewed; photos |

## 4. Network Topology

```
 [Campus WiFi 2.4 GHz — same subnet for all three]

 ESP32 hub ─┐                ┌─► Server (uvicorn :8000, DHCP-reserved)
            ├─ (W) ── (AP) ──┤
 ESP32-CAM ─┘                └─► SSU browsers (LAN / campus network)
```

- All three WiFi actors sit on one subnet so browser→CAM streaming (I7) works without a relay.
- DHCP reservations for hub + CAM + server — static addressing makes `CAM_IP` in firmware and the stream URL in the dashboard stable forever.
- Server can live on a laptop (POC), a campus server, or a small VPS; only requirement is LAN reachability from the pole for photo fetch, and campus reachability for the dashboard.

## 5. Failure Modes & Degradation

| Failure | Immediate effect | System behavior |
|---|---|---|
| WiFi down at pole | upload fails | photos still saved to CAM microSD; hub retries next event; card reconciled at maintenance |
| CAM reboot (heap) | /capture fails | hub logs incident with `confirmed` but no photo; microSD keeps prior photos; CAM auto-recovers ~60 s |
| Server down | POST fails | same as WiFi-down: evidence on microSD, dashboard obviously dark |
| Beam mis-aimed (far post knocked) | confirmed=false always | incidents still log (radar-only); flagged in dashboard for review |
| Radar phantom (branch/banners) | false kph ≥ 5 | beam stays intact → confirmed=false → filtered by SSU |
| Photo too dark (night) | OCR fails | plate_text stays NULL; nightly OCR retry; long-term fix is lighting |

**Design principle:** no single failure loses evidence — the CAM's microSD is the always-write copy, the cloud is the convenient copy.

## 6. Deployment Topology (POC → production path)

| Stage | What runs where |
|---|---|
| POC / evaluation | One laptop: FastAPI + SQLite + dashboard; both boards on bench WiFi |
| Pilot (one lane) | Campus server or mini-PC in a guard post; one pole unit + far post |
| Scale (multi-lane) | Per-lane pole units all POST to one server; SQLite → PostgreSQL; uploads/ → object storage |

The architecture doesn't change between stages — only where the server process runs and which database driver it opens.

---

Interfaces in detail: [BACKEND.md](BACKEND.md) §2 · Firmware contracts: [FIRMWARE.md](FIRMWARE.md) · Signals: [BLOCK-DIAGRAM.md](BLOCK-DIAGRAM.md) · Logic: [FLOWCHART.md](FLOWCHART.md)
