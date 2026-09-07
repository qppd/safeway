# Firmware Guide — Two-Board Architecture

Two independent sketches, one per board. Each flashes over its own USB port.

**Estimated time:** 1.5–2.5 hours (IDE setup + both flashes)

| Sketch | Board | Job |
|---|---|---|
| `safeway-cam` | ESP32-CAM-MB | photo server: `/capture` (JPEG) + `/stream` (live MJPEG) + SD backup |
| `safeway-hub` | ESP32 38-pin | Doppler speed + break-beam confirm + buzzer + fetch photo + upload to API |

---

## 1. Arduino IDE Setup (once, both boards use it)

1. Install **Arduino IDE 2.x** — https://www.arduino.cc/en/software
2. Add the ESP32 board package:
   - *File → Preferences → Additional boards manager URLs*:
     ```
     https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json
     ```
   - *Tools → Board → Boards Manager* → search **esp32** → install **esp32 by Espressif Systems** (v2.x+)
3. Drivers (both are usually automatic on Windows 10/11):
   - **38-pin board:** CP2102 driver — https://www.silabs.com/developer-tools/usb-to-uart-bridge-vcp-drivers
   - **CAM-MB board:** CH340 driver — https://www.wch-ic.com/downloads (only if the port doesn't appear)
4. No extra libraries needed — WiFi, HTTPClient, base64, SD_MMC ship with the ESP32 core.

---

## 2. Sketch 1 — `safeway-cam` (camera board)

Serves one still endpoint and one stream endpoint, and saves every captured photo to microSD as the local backup.

```cpp
/* safeway-cam — ESP32-CAM (AI-Thinker) on MB programmer board
   Endpoints:  /capture  -> single JPEG (also saved to microSD)
               /stream   -> MJPEG live feed (dashboard "Live" pane)
               /         -> tiny status page                                  */

#include "esp_camera.h"
#include <WiFi.h>
#include <SD_MMC.h>

// ---------- CONFIG ----------
const char* WIFI_SSID = "Campus-WiFi";     // 2.4 GHz network!
const char* WIFI_PASS = "********";
// ----------------------------

// AI-Thinker ESP32-CAM pin model
#define PWDN_GPIO_NUM  32
#define RESET_GPIO_NUM -1
#define XCLK_GPIO_NUM   0
#define SIOD_GPIO_NUM  26
#define SIOC_GPIO_NUM  27
#define Y9_GPIO_NUM    35
#define Y8_GPIO_NUM    34
#define Y7_GPIO_NUM    39
#define Y6_GPIO_NUM    36
#define Y5_GPIO_NUM    21
#define Y4_GPIO_NUM    19
#define Y3_GPIO_NUM    18
#define Y2_GPIO_NUM     5
#define VSYNC_GPIO_NUM 25
#define HREF_GPIO_NUM  23
#define PCLK_GPIO_NUM  22

WiFiServer server(80);
int snapCount = 0;

bool camInit() {
  camera_config_t cc = {};
  cc.ledc_channel = LEDC_CHANNEL_0;
  cc.ledc_timer   = LEDC_TIMER_0;
  cc.pin_pwdn     = PWDN_GPIO_NUM;  cc.pin_reset = RESET_GPIO_NUM;
  cc.pin_xclk     = XCLK_GPIO_NUM;  cc.pin_ssc_siod = SIOD_GPIO_NUM;
  cc.pin_ssc_sioc = SIOC_GPIO_NUM;  cc.pin_vsync = VSYNC_GPIO_NUM;
  cc.pin_href     = HREF_GPIO_NUM;  cc.pin_pclk  = PCLK_GPIO_NUM;
  cc.pin_d0 = Y2_GPIO_NUM; cc.pin_d1 = Y3_GPIO_NUM;
  cc.pin_d2 = Y4_GPIO_NUM; cc.pin_d3 = Y5_GPIO_NUM;
  cc.pin_d4 = Y6_GPIO_NUM; cc.pin_d5 = Y7_GPIO_NUM;
  cc.pin_d6 = Y8_GPIO_NUM; cc.pin_d7 = Y9_GPIO_NUM;
  cc.xclk_freq_hz = 20000000;
  cc.pixel_format = PIXFORMAT_JPEG;
  cc.frame_size   = FRAMESIZE_SVGA;      // 800x600 — plate-readable, small upload
  cc.jpeg_quality = 12;                   // lower = better, heavier
  cc.fb_count     = 1;
  cc.grab_mode    = CAMERA_GRAB_LATEST;
  return esp_camera_init(&cc) == ESP_OK;
}

void sdSave(camera_fb_t* fb) {
  String path = "/sw_" + String(millis()) + "_" + String(snapCount++) + ".jpg";
  File f = SD_MMC.open(path, FILE_WRITE);
  if (f) { f.write(fb->buf, fb->len); f.close(); }
}

void setup() {
  Serial.begin(115200);
  if (!camInit()) { Serial.println("CAMERA FAIL"); while (true) delay(100); }
  SD_MMC.begin("/sdcard", true);            // 1-bit mode: leaves GPIOs free
  WiFi.begin(WIFI_SSID, WIFI_PASS);
  while (WiFi.status() != WL_CONNECTED) { delay(200); Serial.print("."); }
  Serial.println("\nCAM ready at http://" + WiFi.localIP().toString());
  server.begin();
}

void loop() {
  WiFiClient c = server.available();
  if (!c) return;
  String req = c.readStringUntil('\r'); c.readStringUntil('\n');
  String path = req.substring(req.indexOf(' ') + 1);
  path = path.substring(0, path.indexOf(' '));

  camera_fb_t* fb = nullptr;
  if (path == "/capture" || path == "/") {
    fb = esp_camera_fb_get();
    if (fb && path == "/capture") sdSave(fb);
  }

  if (path == "/capture" && fb) {           // single JPEG
    c.println("HTTP/1.1 200 OK\nContent-Type: image/jpeg");
    c.println("Content-Length: " + String(fb->len) + "\n");
    c.write(fb->buf, fb->len);
    esp_camera_fb_return(fb);
  }
  else if (path == "/stream") {             // MJPEG live feed
    if (fb) esp_camera_fb_return(fb);
    c.println("HTTP/1.1 200 OK\nContent-Type: multipart/x-mixed-replace; boundary=sw");
    while (c.connected()) {
      fb = esp_camera_fb_get();
      if (!fb) continue;
      c.println("--sw\nContent-Type: image/jpeg");
      c.println("Content-Length: " + String(fb->len) + "\n");
      c.write(fb->buf, fb->len);
      c.println();
      esp_camera_fb_return(fb);
      // ~10 fps cap
    }
  }
  else if (path == "/" && fb) {             // status page
    if (fb) esp_camera_fb_return(fb);
    c.println("HTTP/1.1 200 OK\nContent-Type: text/html\n\n"
              "<h2>SafeWay CAM</h2>/capture /stream OK");
  }
  else c.println("HTTP/1.1 404\nConnection: close\n");
  delay(5); c.stop();
}
```

**Write down the IP it prints** (e.g. `192.168.1.45`) — the hub and the dashboard both need it. For production, give the CAM a **DHCP reservation** on the campus router so it never changes ([DEPLOYMENT.md §pre-install](DEPLOYMENT.md#2-pre-install-checklist)).

---

## 3. Sketch 2 — `safeway-hub` (38-pin sensor board)

The brain: counts Doppler pulses, converts Hz→km/h, confirms with the laser break-beam, buzzes on overspeed, fetches the photo from the CAM, uploads the incident.

```cpp
/* safeway-hub — ESP32 38-pin (sensor hub)
   GPIO 34 = CDM324 OUT | GPIO 25 = laser receiver DO | GPIO 27 = buzzer */

#include <WiFi.h>
#include <HTTPClient.h>
#include <base64.h>
#include <ArduinoJson.h>   // Arduino Library Manager: "ArduinoJson" by Benoit Blanchon

// ---------- CONFIG ----------
const char* WIFI_SSID = "Campus-WiFi";
const char* WIFI_PASS = "********";
const char* CAM_IP    = "192.168.1.45";     // from safeway-cam serial
const char* API_URL   = "http://192.168.1.10:8000/api/incidents";
const float HZ_PER_KPH = 44.7;              // CDM324 @ 24.125 GHz
                                            // (HB100 10.525 GHz variant: 19.49)
const float COSINE_ANGLE_DEG = 0.0;         // set if mounted off-axis (see HARDWARE §8)
const float SPEED_LIMIT_KPH  = 30.0;         // campus limit
const float MIN_SPEED_KPH   = 5.0;          // below this = noise, ignore
const bool  BEAM_BREAKS_LOW = true;         // true: DO LOW = beam intact (most modules)
                                            // set from the bench polarity check (HARDWARE §5.2)
// ----------------------------

#define PIN_RADAR   34
#define PIN_BEAM     25
#define PIN_BUZZER   27

volatile uint32_t pulses = 0;
IRAM_ATTR void onPulse() { pulses++; }

// --- laser break-beam: interrupt + software debounce ---
volatile uint32_t beamEdgeMs = 0;           // last accepted edge time (ms)
volatile bool beamBroken = false;           // true = vehicle (or object) blocking beam
IRAM_ATTR void onBeamEdge() {
  uint32_t now = millis();
  if (now - beamEdgeMs < 50) return;        // debounce: ignore edges <50 ms apart
  beamEdgeMs = now;
  bool level = digitalRead(PIN_BEAM);        // read level at the edge
  // If DO LOW = beam intact (BEAM_BREAKS_LOW true), a rising edge = beam broken;
  // falling edge = beam restored.
  bool broken = BEAM_BREAKS_LOW ? (level == HIGH) : (level == LOW);
  beamBroken = broken;
}

float lastKph = 0, peakKph = 0, peakHz = 0;
uint32_t winStart = 0, lastActiveMs = 0, beamBrokenMs = 0;
bool inEvent = false, beamConfirmed = false;

float kphFromHz(float hz) {
  float measured = hz / HZ_PER_KPH;
  if (COSINE_ANGLE_DEG > 0.1) measured /= cos(COSINE_ANGLE_DEG * PI / 180.0);
  return measured;
}

bool uploadIncident(float kph, float hz, bool confirmed, const String& photoB64) {
  WiFiClient wc; HTTPClient http;
  http.begin(wc, API_URL);
  http.addHeader("Content-Type", "application/json");
  String body = "{\"device_id\":\"safeway-01\",\"speed_kph\":" + String(kph, 1)
              + ",\"limit_kph\":" + String(SPEED_LIMIT_KPH, 0)
              + ",\"doppler_hz\":" + String(hz, 0)
              + ",\"confirmed\":" + (confirmed ? "true" : "false")
              + ",\"photo_b64\":\"" + photoB64 + "\"}";
  int code = http.POST(body);
  http.end();
  return code == 200 || code == 201;
}

String fetchPhotoB64() {                    // GET /capture from the CAM
  WiFiClient wc; HTTPClient http;
  http.begin(wc, String("http://") + CAM_IP + "/capture");
  int code = http.GET();
  if (code != 200) { http.end(); return ""; }
  int len = http.getSize();
  String b64;
  if (len > 0) b64.reserve(base64::encodeLength(len));
  WiFiClient* s = http.getStreamPtr();
  uint8_t buf[512];
  int got;
  while ((got = s->readBytes(buf, sizeof(buf))) > 0)
    b64 += base64::encode(buf, got);
  http.end();
  return b64;
}

void setup() {
  Serial.begin(115200);
  pinMode(PIN_RADAR, INPUT);
  pinMode(PIN_BEAM, INPUT_PULLUP);           // receiver comparator DO (GPIO 25 has pull-ups)
  pinMode(PIN_BUZZER, OUTPUT); digitalWrite(PIN_BUZZER, LOW);
  attachInterrupt(digitalPinToInterrupt(PIN_RADAR), onPulse, RISING);
  attachInterrupt(digitalPinToInterrupt(PIN_BEAM), onBeamEdge, CHANGE);
  // initial beam state (no edge has fired yet)
  delay(100);                                // let the receiver settle
  bool lvl = digitalRead(PIN_BEAM);
  beamBroken = BEAM_BREAKS_LOW ? (lvl == HIGH) : (lvl == LOW);
  WiFi.begin(WIFI_SSID, WIFI_PASS);
  while (WiFi.status() != WL_CONNECTED) { delay(200); Serial.print("."); }
  Serial.println("\nSafeWay HUB ready");
  Serial.printf("beam state at boot: %s\n", beamBroken ? "BROKEN (check alignment)" : "intact");
}

void loop() {
  // --- 300 ms Doppler window ---
  if (millis() - winStart < 300) return;
  uint32_t elapsed = millis() - winStart;
  noInterrupts(); uint32_t p = pulses; pulses = 0; interrupts();
  float hz  = (float)p * 1000.0f / elapsed;
  float kph = kphFromHz(hz);
  winStart = millis();

  if (WiFi.status() != WL_CONNECTED) WiFi.reconnect();

  // --- event state machine ---
  if (kph >= MIN_SPEED_KPH) {
    if (!inEvent) { inEvent = true; peakKph = 0; peakHz = 0; beamConfirmed = false; }
    if (kph > peakKph) { peakKph = kph; peakHz = hz; }
    lastActiveMs = millis();

    // laser break-beam: did a solid object cross the lane during the event?
    if (beamBroken) {
      beamConfirmed = true;
      beamBrokenMs = millis();
      Serial.println("BEAM BROKEN");
    }

    // buzzer: live warn while actively speeding
    digitalWrite(PIN_BUZZER, kph > SPEED_LIMIT_KPH ? HIGH : LOW);

    if (kph > SPEED_LIMIT_KPH) Serial.printf("ACTIVE %.1f km/h (%.0f Hz) beam=%s\n",
                                             kph, hz, beamBroken ? "broken" : "intact");
  }
  else if (inEvent) {
    digitalWrite(PIN_BUZZER, LOW);
    if (millis() - lastActiveMs > 1500) {   // lane clear -> close event
      inEvent = false;
      if (peakKph > SPEED_LIMIT_KPH) {       // VIOLATION
        Serial.printf("VIOLATION peak %.1f km/h | confirmed=%s\n",
                      peakKph, beamConfirmed ? "yes" : "no");
        // confirm beep
        for (int i = 0; i < 2; i++) { digitalWrite(PIN_BUZZER, HIGH); delay(120);
                                     digitalWrite(PIN_BUZZER, LOW);  delay(80); }
        String photo = fetchPhotoB64();
        bool ok = uploadIncident(peakKph, peakHz, beamConfirmed, photo);
        Serial.println(ok ? "uploaded" : "UPLOAD FAILED (photo on CAM microSD)");
      } else {
        Serial.printf("pass: %.1f km/h (under limit)\n", peakKph);
      }
    }
  }
}
```

**Design notes:**

- **Peak-hold logic:** a car is visible to the radar for 2–3 s; each 300 ms window is a sample, and the event's recorded speed is the **peak** — matches how enforcement radar works and gives the fairest reading for a capstone evaluation.
- **Buzzer behavior:** sounds *while* the vehicle is actively over the limit (warns the driver in real time, per the paper's intent) + a confirmation double-beep when the violation is logged.
- **`confirmed` field:** the break-beam broke during the radar event → a solid object physically crossed the lane → guards against radar phantoms (branches, pedestrians). Dashboard shows it; SSU filters on it.
- **Fail-safe:** if the CAM or WiFi is down, the CAM's microSD still holds every photo (`/capture` handler saves before serving) — pull the card during maintenance ([DEPLOYMENT.md §4](DEPLOYMENT.md#4-connectivity-failover-notes)).
- **HB100 variant?** change one constant (`HZ_PER_KPH = 19.49`) — nothing else.

---

## 4. Flashing Procedure

### 4.1 Hub (38-pin board)
1. USB cable → Tools → Board: **ESP32 Dev Module**
2. Upload. Done — CP2102 handles reset/boot automatically.

### 4.2 CAM board (on MB programmer)
1. Seat the ESP32-CAM firmly on the MB board (edge connector, camera ribbon away from USB).
2. USB cable → Tools → Board: **AI Thinker ESP32-CAM**
3. Upload. **The MB board's auto-download circuit handles IO0** — no jumper needed. If the IDE can't find the board: hold the MB's **IO0/BOOT** button, click Upload, release when "Connecting..." appears.
4. Serial Monitor at 115200 → note the **CAM IP address**.

### 4.3 Bench smoke test (before mounting anything)
1. Hub serial shows `ACTIVE` lines when you wave a hand in front of the radar.
2. Browser on the same WiFi: `http://<cam-ip>/stream` → live feed moves.
3. Block/unblock the laser beam with a book at ~1 m → hub prints `BEAM BROKEN`, boot message shows beam state.
4. With the API running ([BACKEND.md](BACKEND.md)): drive a phone-flashlight "event" over the radar at speed → dashboard shows the incident with photo.

---

## 5. Tuning Constants

| Constant | Default | Effect |
|---|---|---|
| `SPEED_LIMIT_KPH` | 30 | violation threshold + buzzer trigger |
| `HZ_PER_KPH` | 44.7 | CDM324 physics — leave unless HB100 (19.49) |
| `COSINE_ANGLE_DEG` | 0 | set to measured mount angle to remove cosine under-read |
| `BEAM_BREAKS_LOW` | true | beam polarity — set from the §5.2 bench check (`true` = DO LOW means beam intact, most modules) |
| `MIN_SPEED_KPH` | 5 | noise floor; raise if tree-wobble false-triggers |
| Window | 300 ms | shorter = faster response, noisier Hz estimate |
| Event close | 1500 ms | silence before closing an event (lane clear) |

Tuning order for the field: `MIN_SPEED_KPH` first (kill phantom triggers), then `BEAM_BREAKS_LOW` from the bench polarity check, then `COSINE_ANGLE_DEG` from your measured install angle.

---

Next: the server that receives it → [BACKEND.md](BACKEND.md)
