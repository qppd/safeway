# Backend & Dashboard Guide — Cloud API

The server side of SafeWay: a REST API that receives violation incidents from the ESP32 hub, stores them, runs plate recognition, and serves the monitoring dashboard (with live lane feed) for SSU personnel.

**Estimated time:** 2–4 hours (first deploy)

---

## Stack (POC-friendly choices)

| Layer | Choice | Why |
|---|---|---|
| API | **FastAPI** (Python 3.11) | Async, automatic OpenAPI docs, multipart/JSON photo handling |
| Database | **SQLite** (file) → upgrade path: PostgreSQL | Zero-config for POC; schema ports cleanly |
| Photos | Local `uploads/` dir, path stored in DB | Simple; swap for S3-compatible storage later |
| Plate recognition | **OpenCV + pytesseract** (default) / OpenALPR (optional upgrade — upstream has no `ph` region) | Runs on captured photos (server-side, not on the ESP32) |
| Dashboard | FastAPI-served HTML + **Tailwind** + vanilla JS (fetch) | Single process to deploy; no separate frontend build |
| Live feed | `<img>` polling the CAM's `/stream` (~1 frame/s) | No server relay needed — browser pulls frames from the CAM over the same network |

Everything runs in one process you can host on a laptop, a campus server, or a small VPS.

---

## Repository Layout (suggested)

```
server/
├── main.py            # FastAPI app + routes + schema + db
├── plates.py          # plate recognition pipeline
├── requirements.txt
├── uploads/           # captured photos (jpg)
└── static/
    └── dashboard.html # SSU monitoring dashboard
```

---

## 1. Database Schema

```sql
CREATE TABLE IF NOT EXISTS incidents (
    id            INTEGER PRIMARY KEY AUTOINCREMENT,
    device_id     TEXT NOT NULL,
    detected_at   TEXT NOT NULL,          -- ISO 8601, server-received time
    speed_kph     REAL NOT NULL,          -- peak speed during the event
    limit_kph     REAL NOT NULL,
    doppler_hz    REAL,                   -- raw Doppler frequency at peak (audit/evidence)
    confirmed     INTEGER DEFAULT 0,      -- 1 = laser break-beam broke during the event (radar+beam agree)
    photo_path    TEXT,                   -- uploads/sw_1234.jpg
    plate_text    TEXT,                   -- from recognition (nullable until processed)
    plate_confidence REAL,
    reviewed      INTEGER DEFAULT 0       -- 0 = new, 1 = acknowledged by SSU
);

CREATE INDEX IF NOT EXISTS idx_incidents_time ON incidents(detected_at DESC);
```

- `doppler_hz` is stored so every speed reading is **auditable**: speed = doppler_hz ÷ 44.7 (÷ cosine correction). If a reading is ever disputed, the raw physics is in the record.
- `confirmed` is the two-sensor agreement flag (radar event + break-beam broken while the vehicle crossed). SSU can filter to confirmed-only for reports; unconfirmed events stay for analysis.
- `detected_at` is stamped **server-side** on receipt (the ESP32 has no reliable clock until you add NTP — see Tuning in [FIRMWARE.md](FIRMWARE.md)).

---

## 2. API Spec

Base URL: `http://<server>:8000` — docs auto-generated at `/docs` (Swagger UI).

### POST `/api/incidents` — receive a violation (called by the ESP32 hub)

Body: `application/json`

| Field | Type | Notes |
|---|---|---|
| `device_id` | string | e.g. `safeway-01` |
| `speed_kph` | float | peak speed, computed on the hub |
| `limit_kph` | float | campus limit |
| `doppler_hz` | float | raw Doppler frequency at peak |
| `confirmed` | bool | laser break-beam broke while the vehicle crossed the lane during the event |
| `photo_b64` | string | base64 JPEG (optional — the CAM's microSD has the backup) |

**201** → `{ "id": 42, "plate_text": null }`
Plate recognition runs in a **background thread** (the hub is answered immediately); the record's plate fields update when OCR completes.

### GET `/api/incidents?limit=50&reviewed=false&confirmed=true` — list for dashboard

Returns newest first: id, time, speed, doppler, confirmed, plate, photo URL, reviewed flag.

### GET `/api/incidents/{id}/photo` — serve the captured JPEG

### PATCH `/api/incidents/{id}` — mark reviewed / annotate

Body: `{ "reviewed": 1, "plate_text": "corrected plate" }` — for SSU corrections when OCR misreads.

### GET `/api/cam-status` — probe the camera board

Server-side `GET http://<cam-ip>/` with 3 s timeout → `{ "cam_online": true }`. Dashboard uses this to gray out the Live pane when the CAM reboots.

---

## 3. Server Implementation (main.py)

```python
import base64, os, sqlite3, threading, uuid
from datetime import datetime, timezone
import requests
from fastapi import FastAPI, HTTPException
from fastapi.responses import FileResponse, HTMLResponse
from pydantic import BaseModel

app = FastAPI(title="SafeWay API", version="0.2.0")
DB = "safeway.db"
UPLOADS = "uploads"
CAM_IP = "192.168.1.45"          # ESP32-CAM address (DHCP-reserved)
os.makedirs(UPLOADS, exist_ok=True)

class Incident(BaseModel):
    device_id: str
    speed_kph: float
    limit_kph: float
    doppler_hz: float | None = None
    confirmed: bool = False
    photo_b64: str | None = None

def db() -> sqlite3.Connection:
    conn = sqlite3.connect(DB)
    conn.row_factory = sqlite3.Row
    return conn

SCHEMA = """
CREATE TABLE IF NOT EXISTS incidents (
    id            INTEGER PRIMARY KEY AUTOINCREMENT,
    device_id     TEXT NOT NULL,
    detected_at   TEXT NOT NULL,          -- ISO 8601, server-received time
    speed_kph     REAL NOT NULL,          -- peak speed during the event
    limit_kph     REAL NOT NULL,
    doppler_hz    REAL,                   -- raw Doppler frequency at peak (audit/evidence)
    confirmed     INTEGER DEFAULT 0,      -- 1 = laser break-beam broke during the event
    photo_path    TEXT,                   -- uploads/sw_1234.jpg
    plate_text    TEXT,                   -- from recognition (nullable until processed)
    plate_confidence REAL,
    reviewed      INTEGER DEFAULT 0       -- 0 = new, 1 = acknowledged by SSU
);
CREATE INDEX IF NOT EXISTS idx_incidents_time ON incidents(detected_at DESC);
"""

with db() as c:  # init schema on boot (same SQL as section 1)
    c.executescript(SCHEMA)

@app.post("/api/incidents", status_code=201)
def create_incident(inc: Incident):
    fname = None
    if inc.photo_b64:
        fname = f"{uuid.uuid4().hex}.jpg"
        with open(os.path.join(UPLOADS, fname), "wb") as f:
            f.write(base64.b64decode(inc.photo_b64))
    now = datetime.now(timezone.utc).isoformat()
    with db() as c:
        cur = c.execute(
            """INSERT INTO incidents
               (device_id, detected_at, speed_kph, limit_kph, doppler_hz,
                confirmed, photo_path)
               VALUES (?,?,?,?,?,?,?)""",
            (inc.device_id, now, inc.speed_kph, inc.limit_kph, inc.doppler_hz,
             int(inc.confirmed), fname))
        new_id = cur.lastrowid
    if fname:  # plate recognition in background (don't block the ESP32)
        threading.Thread(target=run_ocr, args=(new_id, fname), daemon=True).start()
    return {"id": new_id, "plate_text": None, "plate_confidence": None}

def run_ocr(inc_id: int, fname: str):
    from plates import recognize          # see section 4
    try:
        text, conf = recognize(os.path.join(UPLOADS, fname))
        with db() as c:
            c.execute("UPDATE incidents SET plate_text=?, plate_confidence=? WHERE id=?",
                      (text, conf, inc_id))
    except Exception as e:
        print("OCR failed:", e)

@app.get("/api/incidents")
def list_incidents(limit: int = 50, reviewed: bool | None = None,
                   confirmed: bool | None = None):
    q = ("SELECT id, device_id, detected_at, speed_kph, limit_kph, doppler_hz, "
         "confirmed, plate_text, plate_confidence, photo_path, reviewed FROM incidents")
    args = []
    where = []
    if reviewed is not None:
        where.append("reviewed=?"); args.append(int(reviewed))
    if confirmed is not None:
        where.append("confirmed=?"); args.append(int(confirmed))
    if where:
        q += " WHERE " + " AND ".join(where)
    q += " ORDER BY detected_at DESC LIMIT ?"; args.append(limit)
    with db() as c:
        return [dict(r) for r in c.execute(q, args).fetchall()]

@app.get("/api/incidents/{inc_id}/photo")
def photo(inc_id: int):
    with db() as c:
        row = c.execute("SELECT photo_path FROM incidents WHERE id=?", (inc_id,)).fetchone()
    if not row or not row["photo_path"]:
        raise HTTPException(404)
    return FileResponse(os.path.join(UPLOADS, row["photo_path"]))

@app.patch("/api/incidents/{inc_id}")
def update(inc_id: int, body: dict):
    fields, args = [], []
    for k in ("reviewed", "plate_text"):
        if k in body:
            fields.append(f"{k}=?"); args.append(body[k])
    if not fields: raise HTTPException(400, "nothing to update")
    args.append(inc_id)
    with db() as c:
        c.execute(f"UPDATE incidents SET {', '.join(fields)} WHERE id=?", args)
    return {"ok": True}

@app.get("/api/cam-status")
def cam_status():
    try:
        r = requests.get(f"http://{CAM_IP}/", timeout=3)
        return {"cam_online": r.status_code == 200}
    except Exception:
        return {"cam_online": False}

@app.get("/", response_class=HTMLResponse)
def dashboard():
    return open("static/dashboard.html", encoding="utf-8").read()
```

Run: `uvicorn main:app --host 0.0.0.0 --port 8000`

---

## 4. Plate Recognition (plates.py)

The runnable default is pure Python — OpenCV preprocess + Tesseract. OpenALPR is the optional upgrade path, not the default: its agent ships region configs for `us eu au br in kr vn` only — **there is no `ph` region** (`alpr -c ph` errors with "Invalid country") — and the project has been unmaintained since 2018. Real PH-plate ANPR means training a custom region or a cloud service; that's the production upgrade, not the POC.

```python
# Path A (default): pytesseract + OpenCV preprocess
#   pip install pytesseract opencv-python
import cv2, pytesseract, re

def recognize(path: str) -> tuple[str | None, float]:
    img = cv2.imread(path)
    gray = cv2.cvtColor(img, cv2.COLOR_BGR2GRAY)
    gray = cv2.bilateralFilter(gray, 9, 75, 75)
    edges = cv2.Canny(gray, 60, 180)
    text = pytesseract.image_to_string(
        gray, config="--psm 7 -c tessedit_char_whitelist=ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789"
    ).strip()
    m = re.search(r"[A-Z0-9]{5,8}", text.upper())
    return (m.group(0), 0.5) if m else (None, 0.0)
```

```python
# Path B (optional upgrade): OpenALPR agent — only if you train & install a
# PH region config first (upstream regions: us eu au br in kr vn — no 'ph').
import subprocess

def recognize(path: str) -> tuple[str | None, float]:
    out = subprocess.run(
        ["alpr", "-c", "ph", path],          # requires your own trained region
        capture_output=True, text=True, timeout=20
    ).stdout.strip().splitlines()
    if not out: return None, 0.0
    # first line:  -  ABC1234    91.3% confidence
    parts = out[0].split()
    return parts[1], float(parts[2].rstrip("%")) / 100.0
```

> PH plates: 3 letters + 4 digits (private) or LLL-DDDD variants (motor). The regex/whitelist handles both. Low light = garbage OCR — the fix is lighting, not code (see [FIRMWARE.md](FIRMWARE.md) tuning).

---

## 5. Monitoring Dashboard (static/dashboard.html)

Table of recent incidents + photo pane + **live lane feed** from the CAM + filter. Vanilla JS:

```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>SafeWay — SSU Monitoring Dashboard</title>
  <script src="https://cdn.tailwindcss.com"></script>
</head>
<body class="bg-slate-100 p-6">
  <h1 class="text-2xl font-bold mb-4">SafeWay Violations</h1>

  <div class="flex gap-3 mb-4">
    <select id="filter" class="border rounded p-2" onchange="load()">
      <option value="">All</option>
      <option value="false">New only</option>
      <option value="true">Reviewed</option>
    </select>
    <select id="conly" class="border rounded p-2" onchange="load()">
      <option value="">Radar + beam</option>
      <option value="true">Confirmed only</option>
    </select>
    <button onclick="load()" class="bg-blue-600 text-white rounded px-4 py-2">Refresh</button>
  </div>

  <div class="flex gap-6">
    <table id="tbl" class="bg-white rounded shadow text-sm w-2/3">
      <thead class="bg-slate-200"><tr>
        <th class="p-2 text-left">Time</th><th>Speed</th><th>Doppler</th>
        <th>Conf.</th><th>Plate</th><th>Status</th><th>Photo</th></tr></thead>
      <tbody></tbody>
    </table>
    <div class="w-1/3">
      <img id="live" class="rounded shadow w-full bg-slate-300 mb-2" alt="live">
      <div id="livecap" class="text-xs text-slate-500 mb-2">live lane feed</div>
      <img id="photo" class="rounded shadow w-full bg-slate-300" alt="capture">
    </div>
  </div>

<script>
const CAM_IP = "192.168.1.45";           // same as server config
const esc = s => (s ?? '—').replace(/[&<>"']/g,
  c => ({'&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;',"'":'&#39;'}[c]));
const live = document.getElementById('live');
function refreshLive() { live.src = `http://${CAM_IP}/stream?ts=${Date.now()}`; }  // cache-busted poll
refreshLive(); setInterval(refreshLive, 1200);   // pseudo-live ~1 frame/s

async function load() {
  const f = document.getElementById('filter').value;
  const c = document.getElementById('conly').value;
  let url = `/api/incidents?limit=50`;
  if (f) url += `&reviewed=${f}`;
  if (c) url += `&confirmed=${c}`;
  const rows = await (await fetch(url)).json();
  const tb = document.querySelector('#tbl tbody');
  tb.innerHTML = rows.map(i => `
    <tr class="border-t hover:bg-slate-50">
      <td class="p-2">${new Date(i.detected_at).toLocaleString()}</td>
      <td class="text-center font-bold ${i.speed_kph > i.limit_kph * 1.5 ? 'text-red-600' : ''}">
        ${i.speed_kph.toFixed(1)} km/h</td>
      <td class="text-center text-slate-500 font-mono">${i.doppler_hz ? Math.round(i.doppler_hz) + ' Hz' : '—'}</td>
      <td class="text-center">${i.confirmed ? '✓' : '—'}</td>
      <td class="text-center font-mono">${esc(i.plate_text)}</td>
      <td class="text-center">${i.reviewed ? 'Reviewed' : 'New'}</td>
      <td><button onclick="show(${i.id})" class="text-blue-600 underline">view</button></td>
    </tr>`).join('');
  // gray the live pane if the CAM is down
  const st = await (await fetch('/api/cam-status')).json();
  document.getElementById('livecap').textContent =
    st.cam_online ? 'live lane feed' : 'camera offline';
}
async function show(id) {
  document.getElementById('photo').src = `/api/incidents/${id}/photo`;
  await fetch(`/api/incidents/${id}`, {method: 'PATCH',
    headers: {'Content-Type': 'application/json'}, body: '{"reviewed": 1}'});
  load();
}
load(); setInterval(load, 15000);   // auto-refresh 15 s
</script>
</body>
</html>
```

SSU workflow: watch the **live pane** for context → violation rows arrive automatically → click *view* → photo opens, incident marked reviewed. The **Doppler Hz column + Conf. ✓** column give the evidence trail for reports.

> The dashboard must run on the **same network** as the CAM for the live feed (browser polls `http://<cam-ip>/stream` — one JPEG per request). Over VPN this works too; over public internet you'd need a relay — out of POC scope.

---

## 6. Deploy & Security Notes

- **Run behind the campus network/VPN** for the POC. Expose publicly only via a reverse proxy (nginx/caddy) with HTTPS.
- Add a shared API key later: `X-Device-Key` header checked in middleware (the ESP32 hub sets it with `http.addHeader`).
- SQLite is fine for POC volumes; move to PostgreSQL when > 1 device streams.
- Back up `safeway.db` + `uploads/` nightly (the CAM's microSD is the second copy of photos).
- **CORS**: if the dashboard is ever served from a different origin than the API, add `CORSMiddleware` — for POC it's same-origin, nothing to do.

---

Next: calibrate and evaluate → [TESTING.md](TESTING.md)
