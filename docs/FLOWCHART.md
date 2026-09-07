# Flowchart

Runtime behavior, decision by decision — the logic view of SafeWay, matching the actual hub sketch and API.

---

## 1. Hub Main Loop — Speed Event Pipeline

Every 300 ms window, the hub samples, decides, and acts:

```
                        ┌────────────────────┐
                        │  300 ms window end │
                        └─────────┬──────────┘
                                  ▼
                    ┌──────────────────────────┐
                    │ read & reset pulse count  │
                    │ hz = pulses ÷ 0.3 s       │
                    │ kph = hz ÷ 44.7 ÷ cos θ   │
                    └─────────┬────────────────┘
                                  ▼
                        ┌───────────────┐
                   ┌────│ kph ≥ 5.0 ?  │ (MIN_SPEED_KPH noise floor)
                   │ No └──────┬──────┘
                   ▼           │ Yes
            ┌────────────┐     ▼
            │ WiFi okay? │ ┌───────────────────────────┐
            │reconnect if│ │ inEvent? no → start event: │
            │  dropped   │ │ peak=0, beamConfirmed=no   │
            └────────────┘ └─────────┬─────────────────┘
                                    ▼
                          ┌────────────────────┐
                          │ peak-hold: kph>peak│
                          │ → peak=kph, peakHz │
                          └─────────┬──────────┘
                                    ▼
                          ┌────────────────────┐
                          │ beamBroken? → set  │
                          │ beamConfirmed=true │
                          │ (ISR set it)       │
                          └─────────┬──────────┘
                                    ▼
                       ┌─────────────────────┐
                       │ buzzer ON if kph >  │
                       │ SPEED_LIMIT_KPH     │
                       └─────────┬───────────┘
                                 ▼
                    lane quiet > 1500 ms while inEvent?
                   ┌──────── No ── keep sampling ─────────┐
                   ▼ Yes                                  │
        ┌───────────────────────┐                         │
        │ close event           │◄────────────────────────┘
        │ buzzer OFF            │
        └──────────┬────────────┘
                   ▼
          ┌────────────────────┐  no   ┌──────────────────────┐
          │ peak > 30 km/h ?   │──────►│ log "pass, under     │
          └────────┬───────────┘       │ limit", done         │
                   │ yes (VIOLATION)   └──────────────────────┘
                   ▼
        ┌──────────────────────────────┐
        │ double-beep confirm          │
        │ GET /capture from CAM → b64 │
        │ POST /api/incidents {speed, │
        │  limit, doppler_hz,         │
        │  confirmed, photo_b64}      │
        │ ok? uploaded : "CAM microSD │
        │ holds the photo"            │
        └──────────────────────────────┘
```

**Beam ISR (fires any time, GPIO 25, CHANGE):** edge → debounce (<50 ms edges ignored) → read level → `BEAM_BREAKS_LOW` polarity → set/clear `beamBroken`.

## 2. Beam-Confirm vs Radar-Only Event

| Scenario | Radar | Beam | Outcome |
|---|---|---|---|
| Car crosses | kph ≥ 5 | breaks | `confirmed: true` — strongest evidence |
| Branch sways in radar cone | kph ≥ 5 (maybe) | intact | `confirmed: false` — SSU filters it out |
| Car crosses, beam mis-aimed | kph ≥ 5 | intact (fault) | `confirmed: false` — still logged, flagged for review |
| Pedestrian walks the beam | kph < 5 | breaks | no radar event — nothing logged |

## 2b. CAM board sketch loop (simplified)

```
 boot → WiFi join → mDNS/serial print IP → serve:
   GET /capture → grab frame → save JPEG to microSD → return JPEG
   GET /stream  → MJPEG loop: grab frame → send → repeat
   GET /  → status JSON (health check target for /api/cam-status)
```

## 3. Server-Side Incident Pipeline

```
 POST /api/incidents
        │
        ▼
 ┌──────────────────────────────────────────────┐
 │ validate (Pydantic) → decode photo_b64 →      │
 │ save uploads/sw_<uuid>.jpg →                   │
 │ INSERT row: speed, limit, doppler_hz,          │
 │ confirmed, detected_at=now (server clock)      │
 └───────────┬──────────────────────────────────┘
             ▼
 ┌──────────────────────┐   yes   ┌────────────────────────────┐
 │ photo attached?      │────────►│ plate recognition thread:   │
 └──────────┬───────────┘         │ OpenALPR (`alpr -c ph`) →   │
            │ no                  │ fallback: OpenCV preprocess │
            ▼                     │ + pytesseract PSM 7         │
      return 201 {id}             │ → UPDATE plate_text,        │
                                  │   plate_confidence           │
                                  └────────────────────────────┘
```

- Recognition runs inline if it finishes ≤5 s, else queued in a background thread — the hub is never blocked waiting on OCR.
- Nightly OCR retry pass fills in plates for photos that were too dark.

## 4. Dashboard Session (SSU personnel)

```
 open / → GET /api/incidents?reviewed=false
   │
   ├── filter: All / New only / Reviewed
   ├── filter: confirmed only (radar + beam agree)
   ├── click row → GET /api/incidents/{id}/photo → photo pane
   ├── live lane: <img src="http://<cam-ip>/stream"> (direct from CAM)
   ├── review → PATCH /api/incidents/{id} {reviewed: 1}
   └── cam-status badge → GET /api/cam-status (server probes CAM)
```

---

Decision logic source: [FIRMWARE.md](FIRMWARE.md) §3 (hub sketch) · API behavior: [BACKEND.md](BACKEND.md) §2–4
Blocks & signals: [BLOCK-DIAGRAM.md](BLOCK-DIAGRAM.md) · Layers: [SYSTEM-ARCHITECTURE.md](SYSTEM-ARCHITECTURE.md)
