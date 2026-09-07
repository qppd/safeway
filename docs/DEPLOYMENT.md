# Deployment Guide — Site Installation

Taking the validated prototype from the bench to the pilot road: site survey, physical install, connectivity, SSU handover, and maintenance. (PPDIOO **Operate → Optimize**.)

**Estimated time:** half-day install + half-day handover

---

## 1. Site Survey (pick the road segment)

Criteria for the pilot lane — walk candidates with SSU:

| Factor | Requirement | Why |
|---|---|---|
| Lane length | ≥ 30 m straight visible from pole position | Radar needs 10–30 m of approach to build the peak reading |
| Speed history | Known overspeeding complaints | Matches the study's purpose |
| Power | Outlet / posts within extension reach | Two 5V 2A adapters need mains |
| WiFi | RSSI ≥ −70 dBm at the pole (test with a phone at mount height) | Both boards upload reliability |
| Mounting | One rigid post/pole/wall corner at lane edge, box at ~1–1.5 m | Single-pole unit; camera + radar aim |
| Background | No large moving objects in the radar cone: trees, banners, AC condensers, parked cars | Phantom triggers |
| Safety | Lane remains passable; nothing overhanging traffic | Duh |
| SSU visibility | Near a guard post / patrol route | Physical deterrence + maintenance access |

**Record on the survey sheet:** pole GPS/ID, lane width at the crossing point (sets the far-post distance + 22AWG run length), measured radar mount angle (sets `COSINE_ANGLE_DEG`), far-post footing option (existing post/tree/fence vs. new PVC), power outlet location, WiFi SSID + signal, photo of the site from both directions.

## 2. Pre-Install Checklist

- [ ] Bench + calibration tests passed ([TESTING.md](TESTING.md))
- [ ] `BEAM_BREAKS_LOW` polarity verified on the bench (receiver DO level, beam intact vs blocked)
- [ ] `COSINE_ANGLE_DEG` set from the *site* mount angle (or kept 0 if aimed near-parallel)
- [ ] WiFi credentials for the campus SSID configured (2.4 GHz!)
- [ ] `API_URL` points at the production server; `CAM_IP` matches the CAM's **DHCP reservation**
- [ ] **DHCP reservation set on the campus router** for both boards (hub + CAM) — static IPs make everything findable forever
- [ ] Enclosure sealed, desiccant fresh, camera window clean, radar face within 2 cm of the ABS wall
- [ ] microSD formatted, inserted, tested
- [ ] Spare adapter, fuse, zip ties, and screwdriver in the kit box
- [ ] SSU informed of install date/time (traffic marshalling for 1–2 h)

## 3. Physical Install

### 3.1 The pole unit (everything's in one box now)

1. Mount the enclosure on the pole at **1–1.5 m**, camera facing the trigger zone at plate height.
2. **Radar aim:** beam along the traffic direction, **≤15° off the lane axis** — the sweet spot is 10–15°. A protractor app on your phone against the box edge works.
3. **Break-beam alignment (one-time, two-person):** far-post KY-008 dot aimed at the receiver window on the hub enclosure — one person watches the receiver LED / hub serial `BEAM` state, the other nudges the far post until beam reads **intact**. Re-check after typhoons (a knocked far post is the #1 beam failure mode).
4. Power: adapters inside a small weatherproof junction box at the outlet; DC runs up to the enclosure; drip loops at every entry.
5. UV-rated cable ties on all runs; check nothing metallic sits between radar face and road.

### 3.2 Verify, then leave

1. Power up → serial (laptop at the pole) shows WiFi + `SafeWay HUB ready` + `CAM ready`.
2. Browser check: `/stream` live from a phone on campus WiFi.
3. Marshal 3–5 drive-pasts at mixed speeds → all appear on dashboard with photos + live feed works.
4. Verify `confirmed=✓` on at least one drive-past (break-beam agreeing with radar).
5. Physically tug-test every mount and cable.
6. Photograph the finished install for the paper's Figure 8 (prototype deployed).

## 4. Connectivity Failover Notes

- **If site WiFi is flaky:** photos still save to the CAM's microSD — the incident isn't lost, just late to the cloud. The hub retries on next event; during maintenance, pull the card to reconcile ([TESTING.md §4](TESTING.md#4-reliability--soak-test) taught the CAM crash drill — same recovery here).
- **SSU should know: dashboard blank ≠ system dead.** Check the buzzer on a drive-past; if it sounds, the hub is alive and photos are on SD.
- **CAM-only failure:** live feed grays out (`cam-status` shows offline), but speed logging continues — the hub just can't fetch a photo. Fix on the next maintenance window.
- **Hub-only failure:** live feed keeps working (CAM serves the stream independently) — SSU sees the lane but no new records. Buzzer silent on drive-pasts = hub is the issue.

## 5. SSU Handover & Training (Operate)

Half-day session with the personnel who will actually use it:

1. **Dashboard walkthrough** (30 min): live feed pane, filter new, view photo, filter confirmed-only, mark reviewed, export the incident log (paper trail for reports).
2. **Daily routine** (15 min): morning check of overnight violations; reconcile against guard log; glance at the live feed.
3. **What the buzzer means**: overspeed event in progress — no action needed at the device.
4. **Limits of the POC**: single lane, one pole, speed events only — it supplements, not replaces, patrols.
5. **Reporting faults**: single point of contact + the symptoms table below.
6. Sign-off sheet: names, date, version of guide handed over.

Leave printed copies: this guide's §5–7, the dashboard URL, and the fault table.

## 6. Maintenance Schedule (Optimize)

| Cadence | Task |
|---|---|
| Weekly | Pull CAM microSD (or spot check), verify photos exist for the week's dashboard records |
| Weekly | Visual: enclosure seals, cable ties, camera window |
| Monthly | Clean camera window + receiver window + radar-facing ABS wall (outside face); check desiccant |
| Monthly | Backup `safeway.db` + `uploads/` off the server |
| Quarterly | Re-run the 10-pass calibration ([TESTING.md §2](TESTING.md#2-speed-calibration-the-critical-test)) at the site; re-check radar aim angle + re-torque mounts |
| Yearly | Battery-free devices — but plan adapter/cable replacement every 2 typhoon seasons |

### Typhoon season prep

Take-down criteria (SSU decides): storm signal #2 or expected gusts > 60 km/h → unpower, unbolt the box, bring it indoors. One box, two adapters, four screws — **10-minute teardown** (unclip the 22AWG pair at the far post and coil it with the box; the far-post KY-008 stays capped until re-install).

## 7. Fault Symptoms → Actions (leave with SSU)

| Symptom | First response | If unresolved |
|---|---|---|
| Buzzer silent on a fast vehicle | Radar face blocked? (dirt, stickers, bird nest on the box) | Clean wall; IT ticket — check hub serial |
| Live feed grayed out | CAM rebooted — give it 60 s | IT ticket — power-cycle CAM adapter |
| Dashboard no new records, buzzer works | WiFi down at the pole (phone test) | IT ticket — network team |
| Records but `confirmed` always — | Receiver window dirty / far post knocked (beam permanently broken or mis-aimed) | Clean window; re-align far post (§3.1); check `BEAM_BREAKS_LOW` polarity |
| Photos dark/blurry at night | Confirm lane light is on | Add lighting (capstone Optimize phase) |
| Plate column empty | Photo angle changed? | Re-run camera aim (§3.1) — OCR retries nightly |
| Speeds look wrong (low) | Pole knocked to a steeper angle? | Re-measure angle; update `COSINE_ANGLE_DEG` |
| Either board off entirely | Check its adapter at the outlet | Swap adapter (spare in kit) |
| Phantom events, no cars visible | Something moving in the radar cone? | Clear the corridor; raise `MIN_SPEED_KPH` |

## 8. Data & Privacy Notes

- Captured photos contain plates + occasionally people. Dashboard access = SSU authorized staff only; server stays on campus network.
- The **live feed** shows the public lane continuously — same visibility as standing there; still, restrict dashboard access to authorized SSU accounts and keep the server off the public internet.
- Set a retention rule with SSU (suggest: 90 days for reviewed incidents, then archive/purge) — also keeps SQLite lean.
- The capstone disclaimer permits sharing the *system and code* with acknowledgment; captured *data* belongs to the client institution's policy.

---

Build chain complete: [BOM](BOM.md) → [HARDWARE](HARDWARE.md) → [FIRMWARE](FIRMWARE.md) → [BACKEND](BACKEND.md) → [TESTING](TESTING.md) → **DEPLOY**
