# Backend & Dashboard Guide — Cloud API

The server side of SafeWay: a REST API that receives violation incidents from the ESP32-CAM, stores them, runs plate recognition, and serves the monitoring dashboard for SSU personnel.

**Estimated time:** 2–4 hours (first deploy)

---

## Stack (POC-friendly choices)

| Layer | Choice | Why |
|---|---|---|
| API | **FastAPI** (Python 3.11) | Async, automatic OpenAPI docs, multipart/JSON photo handling |
| Database | **SQLite** (file) → upgrade path: PostgreSQL | Zero-config for POC; schema ports cleanly |
| Photos | Local `uploads/` dir, path stored in DB | Simple; swap for S3-compatible storage later |
| Plate recognition | **OpenALPR** or `openalpr` bindings / OpenCV + Tesseract | Runs on captured photos (server-side, not on the ESP32) |
| Dashboard | FastAPI-served HTML + **Tailwind** + vanilla JS (fetch) | Single process to deploy; no separate frontend build |

Everything runs in one process you can host on a laptop, a campus server, or a small VPS.

---

## Repository Layout (suggested)

```
server/
├── main.py            # FastAPI app + routes
├── database.py        # SQLite schema + connection
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
    speed_kph     REAL NOT NULL,
    limit_kph     REAL NOT NULL,
    gate_m        REAL NOT NULL,
    dt_ms         INTEGER NOT NULL,
    photo_path    TEXT,                   -- uploads/sw_1234.jpg
    plate_text    TEXT,                   -- from recognition (nullable until processed)
    plate_confidence REAL,
    reviewed      INTEGER DEFAULT 0       -- 0 = new, 1 = acknowledged by SSU
);

CREATE INDEX IF NOT EXISTS idx_incidents_time ON incidents(detected_at DESC);
```

`detected_at` is stamped **server-side** on receipt (the ESP32 has no reliable clock until you add NTP — see Tuning in [FIRMWARE.md](FIRMWARE.md)).

---

## 2. API Spec

Base URL: `http://<server>:8000` — docs auto-generated at `/docs` (Swagger UI).

### POST `/api/incidents` — receive a violation (called by ESP32-CAM)

Body: `application/json`

| Field | Type | Notes |
|---|---|---|
| `device_id` | string | e.g. `safeway-gate-01` |
| `speed_kph` | float | computed on device |
| `dt_ms` | int | beam-break interval |
| `gate_m` | float | gate separation used |
| `limit_kph` | float | campus limit |
| `photo_b64` | string | base64 JPEG (optional — SD has the backup) |

**201** → `{ "id": 42, "plate_text": "ABC1234", "plate_confidence": 0.91 }`
Plate recognition runs inline if ≤ 5 s, else queued (background thread) and the record is updated.

### GET `/api/incidents?limit=50&reviewed=false` — list for dashboard

Returns newest first: id, time, speed, plate, photo URL, reviewed flag.

### GET `/api/incidents/{id}/photo` — serve the captured JPEG

### PATCH `/api/incidents/{id}` — mark reviewed / annotate

Body: `{ "reviewed": 1, "plate_text": "corrected plate" }` — for SSU corrections when OCR misreads.

---

## 3. Server Implementation (main.py)

```python
import base64, io, os, sqlite3, threading, time, uuid
from datetime import datetime, timezone
from fastapi import FastAPI, HTTPException
from fastapi.responses import FileResponse, HTMLResponse
from fastapi.staticfiles import StaticFiles
from pydantic import BaseModel

app = FastAPI(title="SafeWay API", version="0.1.0")
DB = "safeway.db"
UPLOADS = "uploads"
os.makedirs(UPLOADS, exist_ok=True)

class Incident(BaseModel):
    device_id: str
    speed_kph: float
    dt_ms: int
    gate_m: float
    limit_kph: float
    photo_b64: str | None = None

def db() -> sqlite3.Connection:
    conn = sqlite3.connect(DB)
    conn.row_factory = sqlite3.Row
    return conn

with db() as c:  # init schema on boot
    c.executescript(open("schema.sql").read())

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
               (device_id, detected_at, speed_kph, limit_kph, gate_m, dt_ms, photo_path)
               VALUES (?,?,?,?,?,?,?)""",
            (inc.device_id, now, inc.speed_kph, inc.limit_kph,
             inc.gate_m, inc.dt_ms, fname))
        new_id = cur.lastrowid
    # plate recognition in background (don't block the ESP32)
    if fname:
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
def list_incidents(limit: int = 50, reviewed: bool | None = None):
    q = "SELECT id, device_id, detected_at, speed_kph, limit_kph, dt_ms, plate_text, plate_confidence, photo_path, reviewed FROM incidents"
    args = []
    if reviewed is not None:
        q += " WHERE reviewed=?"; args.append(int(reviewed))
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

@app.get("/", response_class=HTMLResponse)
def dashboard():
    return open("static/dashboard.html", encoding="utf-8").read()
```

Run: `uvicorn main:app --host 0.0.0.0 --port 8000`

---

## 4. Plate Recognition (plates.py)

Two supported paths — try OpenALPR first:

```python
# Path A: OpenALPR agent installed on server
#   (agent reads file -> prints: plate,confidence)
import subprocess

def recognize(path: str) -> tuple[str | None, float]:
    out = subprocess.run(
        ["alpr", "-c", "ph", path],          # '-c ph' region config
        capture_output=True, text=True, timeout=20
    ).stdout.strip().splitlines()
    if not out: return None, 0.0
    # first line:  -  ABC1234    91.3% confidence
    parts = out[0].split()
    return parts[1], float(parts[2].rstrip("%")) / 100.0
```

```python
# Path B: pure-Python fallback (pytesseract + OpenCV preprocess)
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

> PH plates: 3 letters + 4 digits (private) or LLL-DDDD variants (motor). The regex/whitelist handles both. Low light = garbage OCR — the fix is lighting, not code (see FIRMWARE.md tuning).

---

## 5. Monitoring Dashboard (static/dashboard.html)

Single page: table of recent incidents + photo pane + filter. Vanilla JS:

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
  <h1 class="text-2xl font-bold mb-4">🚗 SafeWay Violations</h1>

  <div class="flex gap-3 mb-4">
    <select id="filter" class="border rounded p-2" onchange="load()">
      <option value="">All</option>
      <option value="false">New only</option>
      <option value="true">Reviewed</option>
    </select>
    <button onclick="load()" class="bg-blue-600 text-white rounded px-4 py-2">Refresh</button>
  </div>

  <div class="flex gap-6">
    <table id="tbl" class="bg-white rounded shadow text-sm w-2/3">
      <thead class="bg-slate-200"><tr>
        <th class="p-2 text-left">Time</th><th>Speed</th><th>Plate</th>
        <th>Conf.</th><th>Status</th><th>Photo</th></tr></thead>
      <tbody></tbody>
    </table>
    <img id="photo" class="rounded shadow w-1/3 bg-slate-300" alt="capture">
  </div>

<script>
async function load() {
  const f = document.getElementById('filter').value;
  const r = await fetch(`/api/incidents?limit=50${f ? '&reviewed=' + f : ''}`);
  const rows = await r.json();
  const tb = document.querySelector('#tbl tbody');
  tb.innerHTML = rows.map(i => `
    <tr class="border-t hover:bg-slate-50">
      <td class="p-2">${new Date(i.detected_at).toLocaleString()}</td>
      <td class="text-center font-bold ${i.speed_kph > i.limit_kph * 1.5 ? 'text-red-600' : ''}">
        ${i.speed_kph.toFixed(1)} km/h</td>
      <td class="text-center font-mono">${i.plate_text ?? '—'}</td>
      <td class="text-center">${i.plate_confidence ? (i.plate_confidence * 100).toFixed(0) + '%' : '—'}</td>
      <td class="text-center">${i.reviewed ? '✅' : '🆕'}</td>
      <td><button onclick="show(${i.id})" class="text-blue-600 underline">view</button></td>
    </tr>`).join('');
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

SSU workflow: dashboard auto-refreshes → click a row's *view* → photo opens, incident marked reviewed.

---

## 6. Deploy & Security Notes

- **Run behind the campus network/VPN** for the POC. Expose publicly only via a reverse proxy (nginx/caddy) with HTTPS.
- Add a shared API key later: `X-Device-Key` header checked in middleware (the ESP32 sets it with `http.addHeader`).
- SQLite is fine for POC volumes; move to PostgreSQL when > 1 device streams.
- Back up `safeway.db` + `uploads/` nightly (the SD card in the device is the second copy of photos).

Next: calibrate and evaluate → [TESTING.md](TESTING.md)
