# Flowchart

Runtime behavior, decision by decision — the logic view of SafeWay, matching the actual hub sketch and API. Diagrams are **Mermaid** and render natively on GitHub.

---

## 1. Hub Main Loop — Speed Event Pipeline

A beam-break fast-path rides alongside the window loop on **each board**: the CAM self-triggers its snapshot the instant **beam #2** breaks (it owns both the camera and the trigger), and the hub fires its photo fetch the instant **beam #1** breaks — pulling the CAM's cached beam-moment frame. Every 300 ms window, the hub then samples, decides, and acts:

```mermaid
flowchart TB
    subgraph FAST["FAST PATH — every loop pass, before the window"]
        BB["beam #1 ISR raised the fetch flag"] --> ACT{"active event and<br/>no photo buffered yet ?"}
        ACT -- "yes" --> FETCH["GET /capture from CAM immediately —<br/>serves its cached beam-#2-moment frame<br/>base64 → heap buffer"]
        ACT -- "no — no radar event (pedestrian)<br/>or photo already held" --> DROP["drop the flag"]
    end

    W["300 ms window ends"] --> COUNT["read &amp; reset pulse count<br/>hz = pulses ÷ 0.3 s<br/>kph = hz ÷ 44.7 ÷ cos θ"]
    COUNT --> FLOOR{"kph ≥ 5.0 ?<br/>(MIN_SPEED_KPH noise floor)"}
    FLOOR -- "no" --> WIFI{"WiFi connected?"}
    WIFI -- "no" --> RC["WiFi.reconnect()"]
    RC --> W2["wait for next window"]
    WIFI -- "yes" --> W2
    FLOOR -- "yes" --> INEV{"inEvent ?"}
    INEV -- "no — start event" --> START["peak = 0 · beamConfirmed = false<br/>photoReady = false — fresh buffer"]
    INEV -- "yes" --> PEAK["peak-hold: kph &gt; peak →<br/>peak = kph · peakHz = hz"]
    START --> PEAK
    PEAK --> BEAM{"beamBroken ?<br/>(set by ISR)"}
    BEAM -- "yes" --> CONF["beamConfirmed = true"]
    BEAM -- "no" --> BZ["buzzer ON if kph &gt; SPEED_LIMIT_KPH"]
    CONF --> BZ
    BZ --> QUIET{"lane quiet &gt; 1500 ms<br/>while inEvent ?"}
    QUIET -- "no" --> W2
    QUIET -- "yes — close event" --> CLOSE["buzzer OFF · inEvent = false"]
    CLOSE --> OVER{"peak &gt; 30 km/h ?"}
    OVER -- "no" --> PASS["log pass — under limit"]
    OVER -- "yes — VIOLATION" --> BEEP["double-beep confirm"]
    BEEP --> FB{"photo buffered from<br/>the beam-break fast path ?"}
    FB -- "yes — plate frame" --> POST["POST /api/incidents<br/>speed · limit · doppler_hz · confirmed · photo_b64"]
    FB -- "no — beam never broke<br/>or the snapshot fetch failed" --> FBF["fallback: GET /capture at close —<br/>late frame, car may be past the pole"]
    FBF --> POST
    POST --> UP{"201 ?"}
    UP -- "yes" --> OK["uploaded"]
    UP -- "no" --> SDF["CAM microSD holds the photo"]
```

**Beam ISRs (fire any time, CHANGE):** beam #1 on hub GPIO 25 — edge → debounce (edges under 50 ms apart ignored) → read level → `BEAM_BREAKS_LOW` polarity → set/clear `beamBroken` → if broken, raise the fetch flag (the loop's fast path pulls the CAM's cached photo within milliseconds). Beam #2 on CAM GPIO 21 — same debounce/polarity logic with `CAM_BEAM_BREAKS_LOW` → if broken, the CAM loop runs `beamSnapshot()` itself: grab frame → cache in PSRAM → save to microSD (no HTTP round-trip involved at all).

## 2. Beam-Confirm vs Radar-Only Event

| Scenario | Radar | Beam #1 | Outcome |
|---|---|---|---|
| Car crosses | kph ≥ 5 | breaks | `confirmed: true` — strongest evidence |
| Branch sways in radar cone | kph ≥ 5 (maybe) | intact | `confirmed: false` — SSU filters it out |
| Car crosses, beam #1 mis-aimed | kph ≥ 5 | intact (fault) | `confirmed: false` — still logged, flagged for review |
| Pedestrian walks the beam | kph under 5 | breaks | no radar event — nothing logged |

(Beam #2 never gates `confirmed` — it only triggers the CAM's snapshot. A car crossing the lane breaks both beams in sequence: #2 fires the cached capture, #1 confirms the event.)

## 3. CAM Board Sketch Loop

```mermaid
flowchart TB
    BOOT["boot"] --> WIFJ["join WiFi"]
    WIFJ --> PRINT["print IP on serial"]
    PRINT --> LOOP2{"beam #2 broke ?<br/>(ISR flag)"}
    LOOP2 -- "yes" --> SNAP["beamSnapshot(): grab frame →<br/>cache in PSRAM → save JPEG to microSD"]
    SNAP --> SERVE
    LOOP2 -- "no" --> SERVE{"request ?"}
    SERVE -- "GET /capture" --> CAP{"cached beam photo<br/>fresher than 8 s ?"}
    CAP -- "yes" --> CACHED["serve cached beam-moment JPEG<br/>(plate frame, no fresh grab)"]
    CAP -- "no — beam missed / rebooted" --> LIVE["live frame grab → return JPEG<br/>(old on-demand behavior)"]
    SERVE -- "GET /stream" --> STR["single frame, no SD write<br/>(dashboard re-polls ~1/s)"]
    SERVE -- "GET /" --> STAT["cheap status page, no frame grab<br/>(cam-status just needs the HTTP 200)"]
```

The CAM's snapshot now fires at **beam #2's interrupt** — before any HTTP request exists. The hub's fetch arrives later and simply receives the cached plate frame (`BEAM2_FRESH_MS` window). Fallback: if no fresh cache exists, `/capture` grabs a live frame — never a hard failure.

## 4. Server-Side Incident Pipeline

```mermaid
flowchart TB
    REQ["POST /api/incidents"] --> VAL["validate (Pydantic) →<br/>decode photo_b64 →<br/>save uploads/sw_uuid.jpg"]
    VAL --> INS["INSERT row: speed · limit · doppler_hz<br/>confirmed · detected_at = now (server clock)"]
    INS --> PHOTO{"photo attached ?"}
    PHOTO -- "no" --> R201["return 201 {id, plate_text: null}"]
    PHOTO -- "yes" --> BG["background OCR thread<br/>(hub answered with 201 immediately)"]
    BG --> OCR["plate recognition:<br/>OpenCV preprocess (gray → bilateral → Canny)<br/>→ pytesseract PSM 7"]
    OCR --> UPD["UPDATE plate_text · plate_confidence"]
    UPD --> R201
```

- Recognition always runs in a background thread — the hub is answered 201 immediately; the row's plate fields fill in when OCR completes.
- Nightly OCR retry pass fills in plates for photos that were too dark.

## 5. Dashboard Session (SSU personnel)

```mermaid
flowchart LR
    OPEN["open /"] --> LIST["GET /api/incidents?reviewed=false"]
    LIST --> FILTER["filter: All / New only / Reviewed"]
    FILTER --> CONF["filter: confirmed only<br/>(radar + beam agree)"]
    CONF --> ROW["click row → GET /api/incidents/id/photo<br/>→ photo pane"]
    ROW --> LIVE["live lane: img polls http://CAM_IP/stream<br/>direct from CAM, ~1 frame/s"]
    LIVE --> REVIEW["review → PATCH /api/incidents/id<br/>{reviewed: 1}"]
    REVIEW --> CAMS["cam-status badge → GET /api/cam-status<br/>(server probes CAM)"]
```

---

Decision logic source: [FIRMWARE.md](FIRMWARE.md) §3 (hub sketch) · API behavior: [BACKEND.md](BACKEND.md) §2–4
Blocks & signals: [BLOCK-DIAGRAM.md](BLOCK-DIAGRAM.md) · Layers: [SYSTEM-ARCHITECTURE.md](SYSTEM-ARCHITECTURE.md)
