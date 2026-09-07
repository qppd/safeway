# SafeWay

**An IoT-Based Vehicle Speed Monitoring and Data Logging System for the CLSU Security & Safety Unit**

SafeWay is a low-cost, institutional-scale vehicle speed monitoring system. A 24 GHz Doppler radar measures vehicle speed directly on a campus road; a two-board ESP32 design photographs the vehicle, serves a live lane feed, logs the incident to a cloud database via API, and sounds a buzzer when the speed limit is exceeded. Security personnel monitor everything from a web dashboard — in real time.

> Proof-of-concept capstone project — BS Information Technology, Central Luzon State University (April 2026)

---

## Table of Contents

- [About](#about)
- [How It Works](#how-it-works)
- [Features](#features)
- [Hardware](#hardware)
- [System Architecture](#system-architecture)
- [Repository Structure](#repository-structure)
- [Build Guide](#build-guide)
- [Quick Start (TL;DR)](#quick-start-tldr)
- [Evaluation Criteria](#evaluation-criteria)
- [Documentation Index](#documentation-index)
- [Team](#team)
- [Acknowledgment](#acknowledgment)

## About

Overspeeding inside institutional campuses is monitored manually — by visual observation and paper logging — which is inconsistent and unreliable. Commercial speed-enforcement systems (radar, ANPR cameras) are engineered for highways and priced beyond what small institutions can afford.

SafeWay answers that gap with a **proof-of-concept** built from ~₱1,650 worth of off-the-shelf IoT parts: it measures speed with radar physics, captures evidence, recognizes plate numbers, logs to the cloud, and surfaces violations on a dashboard with a live camera feed — at institutional scale and budget.

Pilot client: **Central Luzon State University — Security & Safety Unit (CLSU SSU)**. Development follows the **PPDIOO** lifecycle (Prepare, Plan, Design, Implement, Operate, Optimize).

## How It Works

```
Speed = Doppler frequency ÷ 44.7 Hz-per-km/h
```

1. A vehicle enters the radar beam of the **CDM324 24 GHz Doppler module** (mounted on a single pole along the lane)
2. The radar's IF output frequency is **directly proportional to speed** — the ESP32 hub counts pulses, converts Hz → km/h, and holds the **peak** reading for the event
3. The **HC-SR04 ultrasonic** watching the trigger zone confirms a vehicle is physically present (rejects phantom radar triggers from branches/pedestrians)
4. If peak speed > limit (e.g. 20 km/h campus limit):
   - **Buzzer** sounds a live warning while the vehicle is over the limit
   - The hub fetches a photo from the **ESP32-CAM** (`/capture`), which also saves it to microSD
   - Incident record (speed, raw Doppler Hz, confirm flag, photo) is **POSTed to the cloud API**
5. SSU personnel watch the **live lane feed** and review violations on the dashboard; plate numbers are read from captured photos for record accuracy

## Features

- **Doppler radar speed detection** — direct physical measurement (no beam posts across the road, no sun-blind receivers, no alignment maintenance)
- **Two-board architecture** — 38-pin ESP32 hub owns all sensors; ESP32-CAM owns imaging. WiFi between them, zero wires
- **Live lane feed** — SSU sees the monitored road in real time from the dashboard
- **Dual-sensor confirmation** — radar + ultrasonic agreement flags each incident (evidence-grade: raw Doppler Hz stored with every record)
- Photo evidence capture with microSD backup logging
- Cloud data logging through REST API integration
- On-site audible overspeed alert (active buzzer)
- Web dashboard for authorized SSU personnel
- Plate number recognition from captured images
- ~₱1,650 prototype cost vs. commercial radar/ANPR systems

## Hardware

Core components (full verified shopping list with Lazada PH links, prices, and ratings: **[docs/BOM.md](docs/BOM.md)**):

| Component | Role |
|---|---|
| CDM324 24 GHz Doppler radar | Speed measurement (IF frequency = 44.7 Hz per km/h) |
| ESP32 38-pin dev board | Sensor hub — pulse counting, HC-SR04 check, buzzer, cloud upload |
| ESP32-CAM + MB programmer board | Camera board — photo snapshots, live MJPEG stream, microSD backup |
| HC-SR04 ultrasonic | Supplementary detection (redundancy) — confirms vehicle presence |
| Active buzzer 5V | Overspeed alert |
| microSD 16GB Class 10 | Local photo backup on the CAM board |
| LM358 op-amp (fallback) | Signal conditioning if the radar variant's IF is weak |
| 5V 2A adapters ×2 + IP68 enclosure + breadboard, jumpers, zip ties | Power, weatherproofing, assembly |

## System Architecture

```
 Road lane ──────────────────────► direction of travel
          (10–30 m radar coverage)

 ┌ ESP32 38-pin HUB (sensors) ──────────────────────┐
 │  • CDM324 OUT ──► GPIO 34   (Doppler pulse count)│
 │  • HC-SR04 ─────► GPIO 26/25 (presence confirm)  │
 │  • Buzzer ──────► GPIO 27    (overspeed alert)    │
 │  • peak speed = Hz ÷ 44.7 ÷ cos(mount angle)      │
 └───────────┬───────────────────────────────────────┘
             │ WiFi — fetch photo, JSON POST
             │            ┌ ESP32-CAM-MB (camera) ────┐
             │            │  • /capture → JPEG + SD save│
             ├───────────►│  • /stream → live MJPEG     │
             │            └────────────┬───────────────┘
             ▼                         │ live feed
 ┌ Cloud API + Database ──────────────┐ │
 │  • incident records + photos       │ │
 │  • plate number recognition        │ │
 └────────────┬───────────────────────┘ │
              ▼                         ▼
 ┌ Monitoring Dashboard (SSU) ─────────────────────┐
 │  live lane feed + violation table + photo pane  │
 └─────────────────────────────────────────────────┘
```

## Repository Structure

```
safeway/
├── README.md               ← you are here
├── docs/
│   ├── BOM.md              ← verified parts list (Lazada PH)
│   ├── HARDWARE.md         ← wiring, power, enclosure assembly
│   ├── FIRMWARE.md         ← both firmware sketches, flashing, tuning
│   ├── BACKEND.md          ← cloud API, database, dashboard
│   ├── TESTING.md          ← calibration & accuracy tests
│   └── DEPLOYMENT.md       ← site installation at CLSU SSU
└── .gitignore              ← repo hygiene rules
```

## Build Guide

The complete build is organized into five stage guides. Follow them in order:

| Step | Guide | What you'll do |
|---:|---|---|
| 1 | **[docs/HARDWARE.md](docs/HARDWARE.md)** | Verify the radar's IF output, wire the hub (radar + HC-SR04 + buzzer), set up the CAM board, assemble into the enclosure |
| 2 | **[docs/FIRMWARE.md](docs/FIRMWARE.md)** | Set up Arduino IDE, flash both boards (camera server + sensor hub), tune speed limit + angles |
| 3 | **[docs/BACKEND.md](docs/BACKEND.md)** | Stand up the cloud API + database + monitoring dashboard with live feed |
| 4 | **[docs/TESTING.md](docs/TESTING.md)** | Bench-test each board, calibrate Doppler accuracy, run the ISO-based evaluation |
| 5 | **[docs/DEPLOYMENT.md](docs/DEPLOYMENT.md)** | Site survey, pole-mount at the pilot road, train SSU users, maintenance plan |

Each guide is self-contained with wiring tables, commands, and checklists.

## Quick Start (TL;DR)

```bash
git clone https://github.com/qppd/safeway.git
```

1. Order parts from [docs/BOM.md](docs/BOM.md) (~₱1,650 recommended build)
2. Wire per [HARDWARE.md](docs/HARDWARE.md) → flash both boards per [FIRMWARE.md](docs/FIRMWARE.md)
3. Launch the API per [BACKEND.md](docs/BACKEND.md), point the hub firmware at it
4. Drive past the pole — verify speed reading + photo + dashboard record + live feed

## Evaluation Criteria

Per the capstone methodology, the prototype is evaluated on:

| Criterion | Measure |
|---|---|
| Accuracy | Correct speed detection and recorded data |
| Response Time | Detection → alert + data transmission latency |
| Reliability | Consistency across repeated tests |
| Usability | Dashboard ease-of-use for SSU personnel |
| Data Accessibility | Real-time record retrieval |

Software quality is assessed against **ISO/IEC 25010** and the IoT architecture against **ISO/IEC 30141**. See [docs/TESTING.md](docs/TESTING.md).

## Documentation Index

| Document | Contents |
|---|---|
| [BOM.md](docs/BOM.md) | Verified Lazada PH parts, prices, ratings, seller links, budget variants |
| [HARDWARE.md](docs/HARDWARE.md) | Two-board pin map, wiring tables, power design, enclosure, radar aiming |
| [FIRMWARE.md](docs/FIRMWARE.md) | Both annotated sketches (camera server + sensor hub), flashing, tuning |
| [BACKEND.md](docs/BACKEND.md) | REST API spec, database schema, dashboard + live feed, plate recognition |
| [TESTING.md](docs/TESTING.md) | Per-board bench tests, Doppler calibration, evaluation protocol + logs |
| [DEPLOYMENT.md](docs/DEPLOYMENT.md) | Site survey checklist, pole mounting, training, maintenance |

## Team

BS Information Technology · Central Luzon State University

- Roise Anthony M. Barles
- Andrei P. Bernardo
- Marcos A. Salazar

Adviser: Ryan L. Bermoza · Department of Information Technology, College of Engineering

## Acknowledgment

Pilot client: **CLSU Security & Safety Unit**. This project follows the PPDIOO network lifecycle framework (Cisco). The full academic manuscript is kept private and is available from the team on request.

Per the capstone disclaimer: *"The project report or any portion thereof including the source code, or any section may be freely copied and distributed provided that the source is acknowledged."*
