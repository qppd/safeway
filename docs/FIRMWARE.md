# Firmware Guide — ESP32-CAM

Setting up, configuring, and flashing the SafeWay speed-monitor firmware.

**Estimated time:** 1–2 hours (including IDE setup)

---

## 1. Arduino IDE Setup

1. Install **Arduino IDE 2.x** — https://www.arduino.cc/en/software
2. Add the ESP32 board package:
   - *File → Preferences → Additional boards manager URLs*:
     ```
     https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json
     ```
   - *Tools → Board → Boards Manager* → search **esp32** → install **esp32 by Espressif Systems** (v2.x or later)
3. Install the **CH340 driver** if Windows doesn't detect the USB-TTL adapter automatically (search "CH340G driver Windows").
4. Libraries (*Tools → Manage Libraries*): **ArduinoJson** (by Benoit Blanchon). WiFi, HTTPClient, SD_MMC, and the ESP32 camera driver ship with the board package.

## 2. Board Settings

| Setting | Value |
|---|---|
| Board | **AI Thinker ESP32-CAM** |
| Partition Scheme | Huge APP (3MB No OTA/1MB SPIFFS) |
| PSRAM | **Enabled** (required for photo capture) |
| Upload Speed | 115200 (drop if upload fails) |

## 3. Configuration Block

Edit these constants at the top of the sketch before flashing:

| Constant | Meaning | Default / example |
|---|---|---|
| `WIFI_SSID` | 2.4 GHz campus/lab WiFi name | `"SafeWay-Test"` |
| `WIFI_PASS` | WiFi password | — |
| `API_URL` | Cloud API endpoint | `http://<server>:8000/api/incidents` |
| `GATE_DISTANCE_M` | **Measured** center-to-center gate separation | `2.5` |
| `SPEED_LIMIT_KPH` | Campus speed limit | `20.0` |
| `BEAM_BLOCKED_STATE` | Receiver OUT level when beam is blocked (from your HARDWARE.md polarity check) | `LOW` |
| `EVENT_TIMEOUT_MS` | Both gates must trigger within this window (guards pedestrians loitering between gates) | `8000` |

> Important: ESP32-CAM connects to **2.4 GHz WiFi only** (802.11 b/g/n). It will not see 5 GHz networks.

## 4. Firmware Sketch

```cpp
/* ================================================================
   SafeWay — IoT vehicle speed monitor (AI-Thinker ESP32-CAM)
   Gates A & B = laser break-beams on GPIO 13 / 12
   Speed = GATE_DISTANCE_M / (tB - tA)  ->  km/h = m/s * 3.6
   Violation => buzzer (GPIO 15) + photo (OV2640)
              => microSD backup + JSON POST to cloud API
   NOTE: microSD runs in 1-bit mode (frees GPIO 12/13)
   ================================================================ */
#include "esp_camera.h"
#include <WiFi.h>
#include <HTTPClient.h>
#include <ArduinoJson.h>
#include <SD_MMC.h>
#include <base64.h>

// ---------------- CONFIG ----------------
const char* WIFI_SSID     = "YOUR_SSID";
const char* WIFI_PASS     = "YOUR_PASS";
const char* API_URL       = "http://192.168.1.50:8000/api/incidents";
const float GATE_DISTANCE_M  = 2.5;    // <-- measure this on site!
const float SPEED_LIMIT_KPH  = 20.0;
const int   BEAM_BLOCKED_STATE = LOW;  // receiver OUT when beam blocked
const unsigned long EVENT_TIMEOUT_MS = 8000;

// ---------------- PINS ----------------
#define PIN_GATE_A 13
#define PIN_GATE_B 12
#define PIN_BUZZER 15

// AI-Thinker ESP32-CAM camera pin map
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

// ---------------- STATE ----------------
volatile unsigned long tA = 0, tB = 0;
volatile int lastA = HIGH, lastB = HIGH;

void IRAM_ATTR gateA_ISR() {
  int s = digitalRead(PIN_GATE_A);
  if (s != lastA && s == BEAM_BLOCKED_STATE && tA == 0) tA = micros();
  lastA = s;
}
void IRAM_ATTR gateB_ISR() {
  int s = digitalRead(PIN_GATE_B);
  if (s != lastB && s == BEAM_BLOCKED_STATE && tB == 0) tB = micros();
  lastB = s;
}

// ---------------- CAMERA ----------------
bool cameraInit() {
  camera_config_t cfg = {};
  cfg.ledc_channel = LEDC_CHANNEL_0;
  cfg.ledc_timer   = LEDC_TIMER_0;
  cfg.pin_pwdn     = PWDN_GPIO_NUM;  cfg.pin_reset = RESET_GPIO_NUM;
  cfg.pin_xclk     = XCLK_GPIO_NUM;  cfg.pin_sscb_sda = SIOD_GPIO_NUM;
  cfg.pin_sscb_scl = SIOC_GPIO_NUM;  cfg.pin_y9 = Y9_GPIO_NUM;
  cfg.pin_y8 = Y8_GPIO_NUM;  cfg.pin_y7 = Y7_GPIO_NUM;  cfg.pin_y6 = Y6_GPIO_NUM;
  cfg.pin_y5 = Y5_GPIO_NUM;  cfg.pin_y4 = Y4_GPIO_NUM;  cfg.pin_y3 = Y3_GPIO_NUM;
  cfg.pin_y2 = Y2_GPIO_NUM;  cfg.pin_vsync = VSYNC_GPIO_NUM;
  cfg.pin_href = HREF_GPIO_NUM;  cfg.pin_pclk = PCLK_GPIO_NUM;
  cfg.xclk_freq_hz = 20000000;
  cfg.pixel_format  = PIXFORMAT_JPEG;
  cfg.frame_size    = FRAMESIZE_SVGA;   // 800x600 — plate-legible, small upload
  cfg.jpeg_quality  = 12;               // lower = better
  cfg.fb_count      = 1;
  cfg.grab_mode     = CAMERA_GRAB_LATEST;
  esp_err_t err = esp_camera_init(&cfg);
  if (err != ESP_OK) { Serial.printf("camera fail 0x%x\n", err); return false; }
  sensor_t* s = esp_camera_sensor_get();
  s->set_vflip(s, 1);                   // flip if mounted upside-down
  return true;
}

String capturePhotoBase64() {
  camera_fb_t* fb = esp_camera_fb_get();
  if (!fb) return "";
  String b64 = base64::encode(fb->buf, fb->len);
  // save backup copy to SD
  static int n = 0;
  String path = "/sw_" + String(millis()) + "_" + String(n++) + ".jpg";
  File f = SD_MMC.open(path, FILE_WRITE);
  if (f) { f.write(fb->buf, fb->len); f.close(); }
  esp_camera_fb_return(fb);
  return b64;
}

// ---------------- SETUP ----------------
void setup() {
  Serial.begin(115200);
  pinMode(PIN_GATE_A, INPUT); pinMode(PIN_GATE_B, INPUT);
  pinMode(PIN_BUZZER, OUTPUT); digitalWrite(PIN_BUZZER, LOW);

  // microSD in 1-bit mode -> frees GPIO 12/13 for the gates
  if (!SD_MMC.begin("/sdcard", true))
    Serial.println("SD init failed (continuing without backup)");

  cameraInit();

  WiFi.mode(WIFI_STA);
  WiFi.begin(WIFI_SSID, WIFI_PASS);
  Serial.print("WiFi");
  while (WiFi.status() != WL_CONNECTED) { delay(300); Serial.print("."); }
  Serial.println(" connected: " + WiFi.localIP().toString());

  attachInterrupt(digitalPinToInterrupt(PIN_GATE_A), gateA_ISR, CHANGE);
  attachInterrupt(digitalPinToInterrupt(PIN_GATE_B), gateB_ISR, CHANGE);
  Serial.println("SafeWay ready. Waiting for vehicles...");
}

// ---------------- LOOP ----------------
void loop() {
  unsigned long a = tA, b = tB;
  if (a && b) {                          // both gates tripped
    float dt_s   = (b - a) / 1e6;
    float kph    = (GATE_DISTANCE_M / dt_s) * 3.6;
    Serial.printf("PASS  dt=%.4fs  speed=%.1f km/h\n", dt_s, kph);

    if (kph > SPEED_LIMIT_KPH) reportViolation(kph, dt_s);
    delay(1500);                         // re-arm cooldown
    tA = 0; tB = 0;
  }
  if (a && !b && millis() - (a / 1000) > EVENT_TIMEOUT_MS) { tA = 0; }  // stale gate A
  if (b && !a && millis() - (b / 1000) > EVENT_TIMEOUT_MS) { tB = 0; }  // stale gate B
  delay(10);
}

// ---------------- REPORT ----------------
void reportViolation(float kph, float dt_s) {
  Serial.println(">>> VIOLATION — capture + upload");
  digitalWrite(PIN_BUZZER, HIGH);
  String photoB64 = capturePhotoBase64();
  digitalWrite(PIN_BUZZER, LOW);         // buzzer duration = capture window

  StaticJsonDocument<96 * 1024> doc;     // sized for ~64KB base64 photo
  doc["device_id"]   = "safeway-gate-01";
  doc["speed_kph"]   = roundf(kph * 10) / 10.0;
  doc["dt_ms"]       = (long)((b - a) / 1000);
  doc["gate_m"]      = GATE_DISTANCE_M;
  doc["limit_kph"]   = SPEED_LIMIT_KPH;
  doc["photo_b64"]   = photoB64;

  if (WiFi.status() == WL_CONNECTED) {
    HTTPClient http;
    http.begin(API_URL);
    http.addHeader("Content-Type", "application/json");
    http.setTimeout(10000);
    int code = http.POST(String(doc.as<String>()));
    Serial.printf("POST /api/incidents -> %d\n", code);
    http.end();
  } else {
    Serial.println("WiFi lost — photo retained on SD for later sync");
  }
}
```

> **Memory note:** `StaticJsonDocument<96 * 1024>` allocates on the stack and works only because PSRAM is enabled. If you hit boot loops, switch to `JsonDocument` (ArduinoJson 7) or reduce `frame_size` to `FRAMESIZE_VGA`.

## 5. Flashing Procedure

1. Bench-wire per [HARDWARE.md](HARDWARE.md#1-programming-setup-temporary-on-the-bench) — including the **GPIO 0 ↔ GND jumper**.
2. Plug in the CH340 → select its COM port (*Tools → Port*).
3. **Upload**. If it stalls with `Connecting........_____`, the GPIO 0 jumper isn't seated or the board isn't in bootloader mode — press the ESP32-CAM's RST (IO0) button after "Connecting..." starts.
4. After `Done uploading`, **remove the GPIO 0 jumper** and press RST.
5. Open **Serial Monitor at 115200** — you should see the WiFi connect and `SafeWay ready`.

## 6. Bench Smoke Test

1. Wave a hand through Gate A's beam, then Gate B's — Serial prints `PASS dt=... speed=... km/h`.
2. Wave *fast* to exceed the limit → buzzer chirps, photo captures, `POST /api/incidents` returns `200` (once [BACKEND.md](BACKEND.md) is running).
3. Confirm the SD backup: power off, pull the card, check for `/sw_*.jpg` files.

## 7. Tuning Field Values

| Symptom | Fix |
|---|---|
| Speed readings erratic | Re-measure `GATE_DISTANCE_M`; re-align beams (see [HARDWARE.md](HARDWARE.md#gate-alignment-procedure)) |
| Phantom passes (no vehicle) | Sunlight flooding the receiver — shroud it with a short black tube (film canister) |
| Missing slow vehicles | Both beams must clear before re-arm — increase the `delay(1500)` cooldown |
| Photos dark (night) | Add a white LED flood on the lane; the OV2640 needs light for plates |
| Upload fails, device reboots | WiFi RSSI weak or API unreachable — check signal at the pole, add the 470 µF cap (see HARDWARE.md power notes) |

Next: stand up the backend → [BACKEND.md](BACKEND.md)
