# Flowchart

Runtime behavior, decision by decision — the logic view of SafeWay, matching the actual hub sketch and API. Diagrams are **Mermaid** and render natively on GitHub.

---

## 1. Hub Main Loop — Speed Event Pipeline

Every 300 ms window, the hub samples, decides, and acts:

```mermaid
flowchart TB
    W["300 ms window ends"] --> COUNT["read &amp; reset pulse count<br/>hz = pulses ÷ 0.3 s<br/>kph = hz ÷ 44.7 ÷ cos θ"]
    COUNT --> FLOOR{"kph ≥ 5.0 ?<br/>(MIN_SPEED_KPH noise floor)"}
    FLOOR -- "no" --> WIFI{"WiFi connected?"}
    WIFI -- "no" --> RC["WiFi.reconnect()"]
    RC --> W2["wait for next window"]
    WIFI -- "yes" --> W2
    FLOOR -- "yes" --> INEV{"inEvent ?"}
    INEV -- "no — start event" --> START["peak = 0 · beamConfirmed = false"]
    INEV -- "yes" --> PEAK["peak-hold: kph &gt; peak →<br/>peak = kph · peakHz = hz"]
    START --> PEAK
    PEAK --> BEAM{"beamBroken ?<br/>(set by ISR)"}
    BEAM -- "yes" --> CONF["beamConfirmed = true"]
    BEAM -- "no" --> BZ["buzzer ON if kph &gt; SPEED_LIMIT_KPH"]
    CONF --> BZ
    BZ --> QUIET{"lane quiet &gt; 1500 ms<br/>while inEvent ?"}
    QUIET -- "no" --> WAIT
    QUIET -- "yes — close event" --> CLOSE["buzzer OFF · inEvent = false"]
    CLOSE --> OVER{"peak &gt; 30 km/h ?"}
    OVER -- "no" --> PASS["log pass — under limit"]
    OVER -- "yes — VIOLATION" --> BEEP["double-beep confirm"]
    BEEP --> CAP["GET /capture from CAM → base64"]
    CAP --> POST["POST /api/incidents<br/>speed · limit · doppler_hz · confirmed · photo_b64"]
    POST --> UP{"201 ?"}
    UP -- "yes" --> OK["uploaded"]
    UP -- "no" --> SDF["CAM microSD holds the photo"]
```

**Beam ISR (fires any time, GPIO 25, CHANGE):** edge → debounce (edges under 50 ms apart ignored) → read level → `BEAM_BREAKS_LOW` polarity → set/clear `beamBroken`.

## 2. Beam-Confirm vs Radar-Only Event

| Scenario | Radar | Beam | Outcome |
|---|---|---|---|
| Car crosses | kph ≥ 5 | breaks | `confirmed: true` — strongest evidence |
| Branch sways in radar cone | kph ≥ 5 (maybe) | intact | `confirmed: false` — SSU filters it out |
| Car crosses, beam mis-aimed | kph ≥ 5 | intact (fault) | `confirmed: false` — still logged, flagged for review |
| Pedestrian walks the beam | kph under 5 | breaks | no radar event — nothing logged |

## 3. CAM Board Sketch Loop

```mermaid
flowchart LR
    BOOT["boot"] --> WIFJ["join WiFi"]
    WIFJ --> PRINT["print IP on serial"]
    PRINT --> SERVE{"request ?"}
    SERVE -- "GET /capture" --> CAP["grab frame → save JPEG to microSD → return JPEG"]
    SERVE -- "GET /stream" --> STR["MJPEG loop: grab → send → repeat"]
    SERVE -- "GET /" --> STAT["status JSON<br/>(cam-status health target)"]
```

## 4. Server-Side Incident Pipeline

```mermaid
flowchart TB
    REQ["POST /api/incidents"] --> VAL["validate (Pydantic) →<br/>decode photo_b64 →<br/>save uploads/sw_uuid.jpg"]
    VAL --> INS["INSERT row: speed · limit · doppler_hz<br/>confirmed · detected_at = now (server clock)"]
    INS --> PHOTO{"photo attached ?"}
    PHOTO -- "no" --> R201["return 201 {id, plate_text: null}"]
    PHOTO -- "yes" --> FAST{"recognition<br/>finishes ≤ 5 s ?"}
    FAST -- "yes — inline" --> OCR["plate recognition:<br/>OpenALPR (alpr -c ph) →<br/>fallback: OpenCV preprocess + pytesseract PSM 7"]
    FAST -- "no" --> BG["queue in background thread"]
    BG --> OCR
    OCR --> UPD["UPDATE plate_text · plate_confidence"]
    UPD --> R201
```

- Recognition runs inline if it finishes within 5 s, else queued in a background thread — the hub is never blocked waiting on OCR.
- Nightly OCR retry pass fills in plates for photos that were too dark.

## 5. Dashboard Session (SSU personnel)

```mermaid
flowchart LR
    OPEN["open /"] --> LIST["GET /api/incidents?reviewed=false"]
    LIST --> FILTER["filter: All / New only / Reviewed"]
    FILTER --> CONF["filter: confirmed only<br/>(radar + beam agree)"]
    CONF --> ROW["click row → GET /api/incidents/id/photo<br/>→ photo pane"]
    ROW --> LIVE["live lane: img src = http://CAM_IP/stream<br/>(direct from CAM)"]
    LIVE --> REVIEW["review → PATCH /api/incidents/id<br/>{reviewed: 1}"]
    REVIEW --> CAMS["cam-status badge → GET /api/cam-status<br/>(server probes CAM)"]
```

---

Decision logic source: [FIRMWARE.md](FIRMWARE.md) §3 (hub sketch) · API behavior: [BACKEND.md](BACKEND.md) §2–4
Blocks & signals: [BLOCK-DIAGRAM.md](BLOCK-DIAGRAM.md) · Layers: [SYSTEM-ARCHITECTURE.md](SYSTEM-ARCHITECTURE.md)
