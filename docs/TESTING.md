# Testing & Calibration Guide

Bench validation, speed-math calibration, and the formal evaluation protocol from the capstone methodology (PPDIOO **Operate** phase, scored against ISO/IEC 25010 and 30141).

**Estimated time:** half-day bench + 1–2 days field

---

## 1. Bench Tests (before going outside)

Run these in order — each gates the next:

| # | Test | Pass criteria |
|---|---|---|
| B1 | Power rails | 5V rail 4.8–5.2 V with WiFi bursting; AMS1117 out 3.2–3.3 V |
| B2 | Beam polarity | Receiver OUT toggles measurably on beam break (value recorded matches `BEAM_BLOCKED_STATE`) |
| B3 | Serial boot | No boot loop; `SafeWay ready` in serial monitor |
| B4 | WiFi + API | `POST /api/incidents` returns 201 on violation |
| B5 | SD backup | Photo exists on card after a triggered event, even with WiFi off |
| B6 | Buzzer | Sounds during the capture window of a violation, silent otherwise |

## 2. Speed Calibration (the critical test)

The system's only math is `speed = distance / Δt`, so calibrate the *measurement chain*, not the sensor.

### Setup

- Tape measure: mark **Gate A** and **Gate B** positions exactly 2.00 m apart (use a round number — you'll verify against a known speed).
- Enter the same value in `GATE_DISTANCE_M`.

### Reference speeds (no radar gun needed)

| Method | Expected speed |
|---|---|
| Walking briskly | 5–8 km/h |
| Jogging | 9–12 km/h |
| Bicycle (steady) | 15–25 km/h |
| Car at idle creep in 1st, no throttle | 8–12 km/h |
| Car, 2,000 RPM in 2nd (flat road) — compute `km/h ≈ RPM × wheel circumference × 60` | known |

The **car at fixed RPM** method gives a true ground-truth: measure wheel circumference (chalk mark → roll once → measure), then expected kph = (RPM/60) × gear-ratio-adjusted revs × circumference × 3.6. Simpler: use a phone GPS speedometer app as reference (±1–2 km/h typical — acceptable for POC).

### Procedure

1. Trip each reference speed through the gates **10× each**.
2. Record displayed vs. reference speed in the log below.
3. Compute error: `error% = (displayed − reference) / reference × 100`.

### Pass thresholds (POC)

| Metric | Threshold |
|---|---|
| Mean absolute error | ≤ **10%** (paper's POC tolerance) |
| Consistency (σ of repeated same-speed runs) | ≤ 3 km/h |
| Detection rate (events captured / passes made) | ≥ **95%** |

If error is a constant multiplicative factor → your `GATE_DISTANCE_M` is wrong (or beams trip late). Fix the constant, not the firmware.

## 3. Response Time

- Time from beam break B → buzzer: should be < 100 ms (interrupt → loop cycle).
- Beam break B → record visible in dashboard: measure with a stopwatch across 5 runs; POC target **≤ 5 s** (photo encode + upload dominate).
- Log both in the template below.

## 4. Reliability / Soak Test

- Run the device continuously for **48 h** (bench, gates intact).
- Trip the gates periodically (hourly, 20 passes over the window).
- Pass: no reboots, no missed events, no WiFi drops > 3 min, SD card has all photos.
- Watch serial for brownout resets — a reset in the log means power rail problems (see [HARDWARE.md](HARDWARE.md#3-power-design)).

## 5. Usability (SSU panel)

Per the methodology, SSU personnel are test users:

1. Recruit 3–5 SSU staff (the actual dashboard users).
2. Task list: *"Open the dashboard. Find today's violations. View the photo of the fastest one. Mark it reviewed."*
3. Time each user; note where they hesitate or ask for help.
4. One-page feedback form: ease-of-use 1–5, confidence in data 1–5.
5. Pass: all users complete tasks unaided in < 3 min; mean ease ≥ 4/5.

## 6. Evaluation Scoring Sheet (ISO/IEC 25010 mapping)

Score each 1–5 across the test battery; these feed the capstone results chapter:

| Criterion (paper) | ISO/IEC 25010 characteristic | Evidence source | Score (1–5) |
|---|---|---|---|
| Accuracy | Functional suitability | §2 calibration error | |
| Response Time | Performance efficiency | §3 timing logs | |
| Reliability | Reliability | §4 soak results | |
| Usability | Usability | §5 SSU panel | |
| Data Accessibility | Maintainability/Portability | Dashboard retrieval live + history query | |

ISO/IEC 30141 (IoT architecture) checklist — yes/no per item:

- [ ] Devices interact through defined interfaces (API contract honored)
- [ ] Data flow: device → network → cloud → dashboard, all observable
- [ ] Scalable to N gates (schema accepts multiple `device_id`s)
- [ ] Failure isolation: device offline ≠ dashboard down; SD backup covers gaps

## 7. Test Log Templates

### Calibration log

```
Date: ____  Gate distance: ____ m  Firmware: ____
Run | Method (ref speed)        | Displayed kph | Ref kph | Error %
 1  | walk (6)                  |               |         |
 2  | walk (6)                  |               |         |
...
Mean |σ| error: ____ %   Detection rate: ____/____ passes
```

### Soak log

```
Start: ____  End: ____ (48 h)
Reboots: ____   Missed events: ____/____   WiFi drops: ____
Longest outage: ____ min   SD photos present: Y/N
```

### Incident log (used for the paper's data)

| Time | Speed (kph) | Δt (ms) | Plate OCR | OCR conf. | Reviewed by |
|---|---|---|---|---|---|
| | | | | | |

## 8. Common Failures → Causes

| Observation | Likely cause | Fix |
|---|---|---|
| Speed always ~½ expected | Only one gate's interrupt fires; event completing on timeout | Check `BEAM_BLOCKED_STATE`, receiver wiring on silent gate |
| Speed wildly high | Δt near zero — both beams break together | Gates too close or one receiver oscillating in sunlight — shroud it |
| Night photos unusable | OV2640 gain maxed, plate blurry | Add lane lighting (HARDWARE.md/FIRMWARE.md notes) |
| OCR < 50% accuracy | Photo angle/distance wrong for plate size | Move camera closer to plate height, or capture at gate B (plate closest) |
| Dashboard misses records | Upload timeout > 10 s on big photos | Lower `frame_size` to VGA, or serve on campus LAN |

Next: install on site → [DEPLOYMENT.md](DEPLOYMENT.md)
