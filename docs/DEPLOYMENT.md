# Deployment Guide — Site Installation at CLSU SSU

Taking the validated prototype from the bench to the pilot road: site survey, physical install, connectivity, SSU handover, and maintenance. (PPDIOO **Operate → Optimize**.)

**Estimated time:** half-day install + half-day handover

---

## 1. Site Survey (pick the road segment)

Criteria for the pilot lane — walk candidates with SSU:

| Factor | Requirement | Why |
|---|---|---|
| Lane length | ≥ 30 m straight | Car reaches steady speed; room for gates + mounting |
| Speed history | Known overspeeding complaints | Matches the study's purpose |
| Power | Outlet / posts within extension reach | 5V 2A adapter needs mains |
| WiFi | RSSI ≥ −70 dBm at the pole (test with a phone at mount height) | ESP32 upload reliability |
| Mounting | Two rigid posts/poles/walls across the lane, ~1 m height | Gate transmitter/receiver pairs |
| Background | No direct afternoon sun into receivers | Sunlight blinds laser receivers |
| Safety | Lane remains passable; nothing overhanging traffic | Duh |
| SSU visibility | Near a guard post / patrol route | Physical deterrence + maintenance access |

**Record on the survey sheet:** post GPS/pole IDs, measured gate separation, power outlet location, WiFi SSID + signal, photo of the site from both directions.

## 2. Pre-Install Checklist

- [ ] Bench + calibration tests passed ([TESTING.md](TESTING.md))
- [ ] `GATE_DISTANCE_M` updated to the *site* measurement
- [ ] WiFi credentials for the campus SSID configured (2.4 GHz!)
- [ ] `API_URL` points at the production server
- [ ] Enclosure sealed, desiccant fresh, camera window clean
- [ ] microSD formatted; SD + photos dir verified
- [ ] Spare fuse/adapter, zip ties, and screwdriver in the kit box
- [ ] SSU informed of install date/time (traffic marshalling for 1–2 h)

## 3. Physical Install

### 3.1 Gates

1. Mount **Gate A transmitter** on the far-side post at bumper/plate height (~0.8–1.2 m), aimed across the lane.
2. Mount **Gate A receiver** directly opposite, same height. Confirm receiver LED lock.
3. Repeat for **Gate B** exactly the measured distance down the lane — mark both posts so nobody "tidies" them apart later.
4. Check: driving/pushing a cart through each beam toggles the receiver (multimeter or serial monitor).

### 3.2 Main enclosure

1. Mount the enclosure on the **Gate B post** (camera faces the approaching plate — plate is nearest at gate B).
2. Cable-tay (UV-rated) all cable runs down the posts; leave a drip loop at every entry so rain tracks off, not in.
3. Power: adapter inside a separate small junction box at the outlet (weatherproof it too), DC run up to the enclosure.
4. Camera: verify the lane fills the frame with the plate readable at the violation trigger point (check with a test photo, then the dashboard).

### 3.3 Verify, then leave

1. Power up → serial (laptop at the pole) shows WiFi + `SafeWay ready`.
2. Marshal 3–5 drive-pasts at mixed speeds → all appear on dashboard with photos.
3. Physically tug-test every mount and cable.
4. Photograph the finished install for the paper's Figure 8 (prototype deployed).

## 4. Connectivity Failover Notes

- If the site WiFi is flaky: photos still save to SD — the incident isn't lost, just late to the cloud. The BOM's parts are cheap enough that a **second ESP32 relay/Mesh** is overkill for POC; instead schedule the server to accept late syncs (manual card pull during maintenance).
- SSU should know: **dashboard blank ≠ system dead**. Check the buzzer on a drive-past; if it sounds, the device is alive and data is on SD.

## 5. SSU Handover & Training (Operate)

Half-day session with the personnel who will actually use it:

1. **Dashboard walkthrough** (30 min): open dashboard, filter new, view photo, mark reviewed, export the incident log (paper trail for reports).
2. **Daily routine** (15 min): morning check of overnight violations; reconcile against guard log.
3. **What the buzzer means**: overspeed event captured — no action needed at the device.
4. **Limits of the POC**: single lane, one device, speed events only — it supplements, not replaces, patrols.
5. **Reporting faults**: single point of contact + the symptoms table below.
6. Sign-off sheet: names, date, version of guide handed over.

Leave printed copies: this guide's §5–6, the dashboard URL, and the fault table.

## 6. Maintenance Schedule (Optimize)

| Cadence | Task |
|---|---|
| Weekly | Pull SD card (or spot check), verify photos exist for the week's dashboard records |
| Weekly | Visual: beam alignment (LED lock), enclosure seals, cable ties |
| Monthly | Clean camera window + laser apertures; check desiccant (replace if saturated) |
| Monthly | Backup `safeway.db` + `uploads/` off the server |
| Quarterly | Re-run 5-pass calibration ([TESTING.md §2](TESTING.md#2-speed-calibration-the-critical-test)) at the site; re-torque mounts |
| Yearly | Battery-free device — but plan adapter/cable replacement every 2 typhoon seasons |

### Typhoon season prep

Take-down criteria (SSU decides): storm signal #2 or expected gusts > 60 km/h → unpower, unbolt gates, bring enclosure indoors. Gates are 2 zip ties + 4 screws by design — 10-minute teardown.

## 7. Fault Symptoms → Actions (leave with SSU)

| Symptom | First response | If unresolved |
|---|---|---|
| Buzzer silent on a fast vehicle | Check receiver LEDs (sunlight drift?) | Re-aim beams (§3.1) |
| Dashboard no new records, buzzer works | WiFi down at the pole (phone test) | IT ticket — network team |
| Photos dark/blurry at night | Confirm lane light is on | Add lighting (capstone Optimize phase) |
| Plate column empty | Photo angle changed? | Re-run camera aim (§3.2) — OCR retries nightly |
| Device off entirely | Check adapter at the outlet | Swap adapter (spare in kit) |
| Speeds look wrong | Recent pole/mount movement? | Re-measure gates; update `GATE_DISTANCE_M` |

## 8. Data & Privacy Notes

- Captured photos contain plates + occasionally people. Dashboard access = SSU authorized staff only; server stays on campus network.
- Set a retention rule with SSU (suggest: 90 days for reviewed incidents, then archive/purge) — this also keeps SQLite lean.
- The capstone disclaimer permits sharing the *system and code* with acknowledgment; captured *data* belongs to the client institution's policy.

---

Build chain complete: [BOM](BOM.md) → [HARDWARE](HARDWARE.md) → [FIRMWARE](FIRMWARE.md) → [BACKEND](BACKEND.md) → [TESTING](TESTING.md) → **DEPLOY**
