# Testing & Calibration Guide

Bench validation, Doppler calibration, and the formal evaluation protocol from the capstone methodology (PPDIOO **Operate** phase, scored against ISO/IEC 25010 and 30141).

**Estimated time:** half-day bench + 1–2 days field

---

## 1. Bench Tests (before going outside)

Run in order — each gates the next. With two boards, several tests are per-board. Before B1, eyeball the build against the [full wiring diagram](../wiring/circuit_image.png).

| # | Test | Pass criteria |
|---|---|---|
| B1 | Power rails (both boards) | Each 5V rail 4.8–5.2 V under WiFi load; hub rail stays flat when CAM reboots |
| B2 | Radar IF verification | [HARDWARE.md §3](HARDWARE.md#3-if-signal-verification-do-this-first) — hand-wave gives 100–2,000 Hz waveform on serial; idle is flat |
| B3 | Hub boot | No boot loop; `SafeWay HUB ready` in serial monitor |
| B4 | CAM boot | `CAM ready at http://<ip>` printed; `/capture` returns a JPEG in a browser |
| B5 | Live frame | `/stream` returns a JPEG in a browser; the dashboard's live pane refreshes ~1/s |
| B6 | Break-beam #1 sanity | Block the beam with a book at ~1 m → hub prints `BEAM BROKEN`; remove → intact; boot line shows beam state |
| B6b | Break-beam #2 sanity (CAM) | Block beam #2 → CAM serial prints `BEAM2 SNAP → SD ok` and a new `/sw_*.jpg` appears on the microSD; remove → intact |
| B7 | Buzzer | Sounds while speed > limit during an event; silent otherwise |
| B8 | WiFi + API | `POST /api/incidents` returns 201 on a simulated violation |
| B9 | SD backup | Photo exists on the CAM's microSD after an event, even with the API down |
| B10 | Beam-break snapshot | With `SPEED_LIMIT_KPH` temporarily set to 5: open a radar event (wave over the radar), then block beam #2 → CAM caches the frame; the hub's fetch (block beam #1 too) returns the **cached beam-moment JPEG** — hub prints `SNAPSHOT: ok` **within ~1 s of the block** (not at event close); the logged incident carries the photo |
| B11 | Stale-cache fallback | Wait > `BEAM2_FRESH_MS` (8 s) after a beam-#2 break, then request `/capture` → CAM serves a **live grab**, not the stale plate frame |
| B12 | UPS transfer (per [HARDWARE.md §5.5](HARDWARE.md#55-battery-backup-ups-option--one-18650-per-supply), each supply) | Pull mains while under load: rail holds ≥ 4.9 V, hub serial keeps counting Hz, CAM stream doesn't drop; mains back → TP4056 charging LED lights |
| B13 | UPS runtime soak (each hub + CAM rail, once per device before acceptance) | Run the rail on battery only for 4 h under normal load: still ≥ 4.9 V at hour 4, hub logs events the whole time, cell recharges fully overnight afterwards |

## 2. Speed Calibration (the critical test)

The measurement chain is: **Doppler Hz ÷ 44.7 = km/h** (+ cosine correction). Calibrate against ground truth — a phone GPS speedometer app (±1–2 km/h, acceptable for POC) or a car at fixed RPM.

### Setup

- Radar mounted at its intended pole position/angle (bench rig works if it can see the test lane)
- Reference: phone GPS speedometer held by the passenger, **or** bicycle with a cycling computer, **or** car at fixed RPM (compute kph = RPM ÷ 60 × gear-adjusted wheel revs × circumference × 3.6 for a true ground truth)

### Reference speeds (no radar gun needed)

| Method | Expected speed |
|---|---|
| Walking briskly | 5–8 km/h |
| Jogging | 9–12 km/h |
| Bicycle (steady) | 15–25 km/h |
| Car at idle creep in 1st, no throttle | 8–12 km/h |
| Phone GPS speedometer (any vehicle) | live reference |

> Human/bike targets reflect the radar weakly (small radar cross-section) — a **car** gives the strongest, cleanest Doppler tone. Do the primary calibration with the car; use walk/bike passes only for coarse sanity.

### Procedure

1. Drive the car past the radar **10× at ~3 steady speeds** (e.g. ~15, ~30, ~45 km/h by GPS).
2. Record displayed peak vs GPS reference per pass in the log below.
3. Compute error: `error% = (displayed − reference) / reference × 100`.

### Pass thresholds (POC)

| Metric | Threshold |
|---|---|
| Mean absolute error | ≤ **10%** (paper's POC tolerance) |
| Consistency (σ of repeated same-speed runs) | ≤ 3 km/h |
| Detection rate (events captured / passes made) | ≥ **95%** |

### Interpreting systematic errors

| Pattern | Cause | Fix |
|---|---|---|
| Error constant multiplicative (~all reads −5%) | Cosine error — radar angled off-axis | Measure the mount angle; set `COSINE_ANGLE_DEG` |
| Reads ~½ expected | Radar module variant with different IF scaling (or LM358 amp clipping) | Re-run [HARDWARE.md §3](HARDWARE.md#3-if-signal-verification-do-this-first); if constant factor confirmed, recalibrate `HZ_PER_KPH` and document it |
| Random ±30% | Weak IF, noisy trigger | Check the LM358 conditioner build ([HARDWARE.md §4](HARDWARE.md#4-lm358-signal-conditioner-radar-chain)) — verify the 2.5 V bias node and re-run the §3 hand-wave test |

## 3. Response Time

- Radar detects overspeed → buzzer ON: ≤ ~400 ms worst case (a reading only exists at each 300 ms window close — the window is the resolution).
- Event close (lane clear + 1.5 s) → record visible in dashboard: stopwatch 5 runs; POC target **≤ 10 s** (the POST dominates — the photo is already buffered from the beam-break snapshot; sub-5 s on campus LAN).
- Beam-break → `SNAPSHOT: ok` on serial: ≤ ~1 s from the block (the CAM's self-triggered capture is near-instant; the hub's fetch of the cached frame follows within the same window).
- Log both in the template below.

## 4. Reliability / Soak Test

- Run both boards continuously for **48 h** (bench, aimed at a walkway with occasional traffic).
- Pass: no reboots, no missed events, no WiFi drops > 3 min, CAM microSD has all photos, hub serial shows no brownout resets.
- **Two-board specific:** power-cycle the CAM mid-soak (simulating a camera crash). Hub must keep measuring; the next violation photo fetch reconnects automatically. Dashboard's `cam-status` should flip to offline and back.
- Watch hub serial for brownout resets — a reset means power rail problems ([HARDWARE.md §5.4](HARDWARE.md#54-power-design--four-independent-supplies)).
- With the UPS fitted, add a 4 h battery-only run per hub + CAM rail (bench test B13, [HARDWARE.md §5.5](HARDWARE.md#55-battery-backup-ups-option--one-18650-per-supply)) — one full outage per device before acceptance.

## 5. Usability (SSU panel)

Per the methodology, SSU personnel are test users:

1. Recruit 3–5 SSU staff (the actual dashboard users).
2. Task list: *"Open the dashboard. Check the live feed is working. Find today's violations. View the photo of the fastest one. Filter to confirmed-only. Mark it reviewed."*
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

- [ ] Devices interact through defined interfaces (hub↔CAM HTTP, hub↔API JSON contract)
- [ ] Data flow: device → network → cloud → dashboard, all observable
- [ ] Scalable to N poles (schema accepts multiple `device_id`s; each pole = 1 hub + 1 CAM)
- [ ] Failure isolation: CAM down ≠ hub down ≠ dashboard down; microSD backup covers photo gaps

## 7. Test Log Templates

### Calibration log

```
Date: ____  Mount angle: ____°  Firmware: ____  Reference: ____
Run | Method (ref speed)        | Displayed peak kph | Ref kph | Error %
 1  | car @ GPS 15 km/h         |                    |         |
 2  | car @ GPS 15 km/h         |                    |         |
...
Mean |σ| error: ____ %   Detection rate: ____/____ passes   Confirmed rate: ____/____
```

### Soak log

```
Start: ____  End: ____ (48 h)
Reboots (hub/CAM): ____/____   Missed events: ____/____   WiFi drops: ____
Longest outage: ____ min   SD photos present: Y/N   CAM power-cycle recovery: Y/N
```

### UPS runtime log (B13 — one per device)

```
Device/Pole: ____   Rail (hub / CAM / TX#1 / TX#2): ____   Date: ____
Start: ____   End: ____ (4 h)
Cell voltage start: ____ V   At hour 4: ____ V   Rail at hour 4: ____ V
Events logged during outage: ____   Transfer clean (B12): Y/N   Full recharge overnight: Y/N
```

### Incident log (used for the paper's data)

| Time | Speed (kph) | Doppler Hz | Confirmed | Plate OCR | OCR conf. | Reviewed by |
|---|---|---|---|---|---|---|
| | | | | | | |

## 8. Common Failures → Causes

| Observation | Likely cause | Fix |
|---|---|---|
| Serial shows Hz but speed ~½ expected | Radar variant IF scaling differs | Re-verify §3; recalibrate `HZ_PER_KPH`, document |
| Constant −5–10% on everything | Cosine error from mount angle | Measure angle; set `COSINE_ANGLE_DEG` |
| Phantom events with no vehicle | Branches/banners in beam cone; `MIN_SPEED_KPH` too low | Clear the beam corridor; raise the noise floor |
| `confirmed` never true | Beam #1 mis-aimed / DO divider missing / `BEAM_BREAKS_LOW` polarity wrong | Re-align far post; check §5.2 level; flip the polarity constant |
| CAM never prints `BEAM2 SNAP` | Beam #2 mis-aimed / its far-post supply dead / `CAM_BEAM_BREAKS_LOW` polarity wrong | Re-align TX #2; check supply #2; flip the CAM polarity constant |
| Photos serve but show the wrong car (stale plate frame) | `BEAM2_FRESH_MS` too long for the traffic pattern | Lower it (e.g. 5 s); B11 validates the stale-cache fallback |
| Hub can't fetch photo | CAM IP changed (DHCP) | Set DHCP reservation; update `CAM_IP` |
| Photos dark/blurry at night | OV5640 gain maxed, plate unreadable | Add lane lighting (Optimize phase); OCR retries nightly |
| OCR < 50% accuracy | Photo angle/distance wrong for plate size | Camera closer to plate height; capture at trigger zone |
| Dashboard misses records | Upload timeout on big photos | Lower CAM `frame_size` to VGA, or serve on campus LAN |
| CAM stream freezes | CAM heap fragmentation after days | CAM reboots nightly (add `ESP.restart()` at 02:00 in CAM sketch) |
| `SNAPSHOT: failed` at beam-break | CAM rebooting / wrong `CAM_IP` | close-time fallback retries once; verify CAM IP + `cam-status` |
| Rail sags the instant mains is pulled | MT3608 output set below rail voltage · SS34 reversed · cell uncharged | Re-set MT3608 to 5.0 V no-load; check SS34 band faces the rail; charge the cell fully before install |
| Battery bank drains in days with mains present | SS34 missing/backwards — battery branch backfeeds through the charger | Fit/verify SS34 on the **mains branch only**; confirm no voltage at TP4056 OUT with mains on |
| Hub speed readings go erratic during outages | Rail cap too small for the transfer instant | Keep the 470–1000 µF rail cap; re-run B12 |

Next: install on site → [DEPLOYMENT.md](DEPLOYMENT.md)
