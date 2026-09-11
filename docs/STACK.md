# Stack

Everything SafeWay is built on — deliberately small, POC-friendly, and free.

---

## Firmware (ESP32 boards)

| Layer | Choice | Notes |
|---|---|---|
| IDE | **Arduino IDE 2.x** | One IDE for both boards — https://arduino.cc/en/software |
| Toolchain | **esp32 by Espressif Systems** (Arduino core v2.x+) | Board Manager URL: `https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json` |
| Board (hub) | **ESP32 Dev Module** (38-pin, CP2102 USB-UART) | Sensor hub sketch `safeway-hub` — radar pulse counting, break-beam ISR + beam-break photo snapshot, buzzer, upload |
| Board (camera) | **ESP32S3 Dev Module** (ESP32-S3 WROOM N16R8 CAM; CH343P USB-UART + native USB-OTG) | Camera sketch `safeway-cam` — `/capture` JPEG, `/stream` polled live frame, microSD backup. IDE: Flash 16MB · PSRAM: OPI PSRAM |
| Drivers | CP2102 (Silabs VCP) · CH343 (WCH) | Only if the COM port doesn't appear |
| Libraries | WiFi · HTTPClient · base64 · SD_MMC *(bundled with core)* + **ArduinoJson** (Benoit Blanchon, via Library Manager) | Nothing else to install |

## Backend (server)

| Layer | Choice | Why |
|---|---|---|
| Runtime | **Python 3.11** | |
| API framework | **FastAPI** | Async, auto OpenAPI docs at `/docs`, clean JSON/multipart photo handling |
| Server | **uvicorn** | `uvicorn main:app --host 0.0.0.0 --port 8000` — one process |
| Models | **Pydantic** | Request validation (ships with FastAPI) |
| Database | **SQLite** (file `safeway.db`) | Zero-config for POC; schema ports cleanly to **PostgreSQL** when volume grows |
| Photos | Local `uploads/` dir, path in DB | Swap for S3-compatible storage later |
| Plate recognition (default) | **pytesseract** + **OpenCV** (`opencv-python`) | Pure-Python: grayscale → bilateral filter → Canny → Tesseract PSM 7 |
| Plate recognition (upgrade) | **OpenALPR** agent | Only with a custom-trained PH region — upstream ships us/eu/au/br/in/kr/vn only (no `ph`), unmaintained since 2018 |
| Outbound HTTP | **requests** | CAM health check from the server |

## Frontend (dashboard + live feed)

| Layer | Choice | Why |
|---|---|---|
| Page | Server-served static HTML (`static/dashboard.html`) | No SPA framework, no build step — FastAPI serves it directly |
| Styling | **Tailwind CSS** (CDN) | Utility classes, zero toolchain |
| Logic | Vanilla **JS** (`fetch` → API) | List/filter/review incidents, photo popups |
| Live feed | `<img src="http://<cam-ip>/stream">` polled ~1/s | Browser pulls frames straight from the CAM over the LAN — no server relay |

## Hardware (context — see [BOM.md](BOM.md) & [HARDWARE.md](HARDWARE.md))

| Part | Role |
|---|---|
| CDM324 24 GHz Doppler radar | Speed measurement (Doppler IF = 44.7 Hz per km/h) |
| KY-008 laser TX + photodetector receiver | Break-beam presence confirmation across the lane |
| ESP32 38-pin dev board | Sensor hub (radar GPIO 34, beam GPIO 25, buzzer GPIO 27) |
| ESP32-S3 WROOM N16R8 CAM (OV5640) | Photo evidence, live stream, microSD failover |

---

## Why this stack

- **One language on the server** — Python end-to-end: FastAPI routes, Pydantic models, OCR pipeline, dashboard serving.
- **One process to deploy** — uvicorn serves API + dashboard; SQLite is a file; no Docker required for the POC (add it later for the campus server).
- **No frontend build chain** — Tailwind via CDN + vanilla JS means nothing to compile, nothing to break during a capstone timeline.
- **Graceful upgrades** — SQLite → PostgreSQL, local `uploads/` → object storage, local OCR → cloud ANPR: each swap is isolated to one module.
- **All free & open-source** — total software cost ₱0; the budget lives in hardware (~₱1,900).
