# SafeWay 🚗⚡

**An IoT-Based Vehicle Speed Monitoring and Data Logging System for the CLSU Security & Safety Unit**

SafeWay is a low-cost, institutional-scale vehicle speed monitoring system. Two laser break-beam gates measure vehicle speed on campus roads; an ESP32-CAM photographs the vehicle, logs the incident to a cloud database via API, and sounds a buzzer when the speed limit is exceeded. Security personnel monitor everything from a web dashboard.

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

SafeWay answers that gap with a **proof-of-concept** built from ~₱2,500 worth of off-the-shelf IoT parts: it detects speed, captures evidence, recognizes plate numbers, logs to the cloud, and surfaces violations on a dashboard — at institutional scale and budget.

Pilot client: **Central Luzon State University — Security & Safety Unit (CLSU SSU)**. Development follows the **PPDIOO** lifecycle (Prepare, Plan, Design, Implement, Operate, Optimize).

## How It Works

```
Speed = Gate Distance / Time between beam breaks
```

1. A vehicle breaks **Laser Gate A** (KY-008 transmitter + receiver pair) → timestamp `tA`
2. The vehicle breaks **Laser Gate B**, placed 2–3 m down the lane → timestamp `tB`
3. ESP32-CAM computes speed: `(tB − tA)` over the known gate distance
4. If speed > limit (e.g. 20 km/h campus limit):
   - **Buzzer** sounds a warning
   - **OV2640 camera** captures a photo (saved to microSD + uploaded)
   - Incident record (speed, timestamps, photo) is **POSTed to the cloud API**
5. SSU personnel view violations on the **monitoring dashboard**; plate numbers are read from captured photos for record accuracy

## Features

- 🚀 Automatic speed detection via dual laser break-beam gates (no manual radar gun)
- 📸 Photo evidence capture with microSD backup logging
- ☁️ Cloud data logging through REST API integration
- 🔔 On-site audible overspeed alert (active buzzer)
- 🖥️ Web dashboard for authorized SSU personnel
- 🔢 Plate number recognition from captured images
- 💰 ~₱2,500 prototype cost vs. commercial radar/ANPR systems

## Hardware

Core components (full verified shopping list with Lazada PH links, prices, and ratings: **[docs/BOM.md](docs/BOM.md)**):

| Component | Role |
|---|---|
| ESP32-CAM (AI-Thinker, OV2640 2MP) | Main controller — speed timing, photo capture, WiFi uplink |
| KY-008 laser transmitter ×2 | Break-beam emitters (Gates A & B) |
| Laser receiver module ×2 | Break-beam detectors |
| CH340 USB-to-TTL adapter | Programming the ESP32-CAM |
| microSD 16GB Class 10 | Local photo/log storage |
| Active buzzer 5V | Overspeed alert |
| HC-SR04 ultrasonic | Supplementary detection (redundancy) |
| AMS1117 3.3V regulator | Clean 3.3V sensor rail |
| 5V 2A adapter + IP68 enclosure + breadboard, jumpers, zip ties | Power, weatherproofing, assembly |

## System Architecture

```
 [Gate A: KY-008 ──beam──► Receiver]──┐
                                     │ GPIO interrupts + micros() timestamps
 [Gate B: KY-008 ──beam──► Receiver]──┤
                                     ▼
        ┌──────────────────────────────────────┐
        │ ESP32-CAM (OV2640 + microSD + WiFi)  │
        │  • speed = distance / Δt             │
        │  • photo capture on violation         │
        │  • buzzer trigger                     │
        └──────────────┬───────────────────────┘
                       │ HTTPS/HTTP JSON POST (WiFi)
                       ▼
        ┌──────────────────────────────────────┐
        │ Cloud API + Database                  │
        │  • incident records + photos          │
        │  • plate number recognition           │
        └──────────────┬───────────────────────┘
                       ▼
        ┌──────────────────────────────────────┐
        │ Monitoring Dashboard (SSU personnel)  │
        └──────────────────────────────────────┘
```

## Repository Structure

```
safeway/
├── README.md               ← you are here
├── docs/
│   ├── BOM.md              ← verified parts list (Lazada PH)
│   ├── HARDWARE.md         ← wiring, power, enclosure assembly
│   ├── FIRMWARE.md         ← ESP32-CAM code, flashing, configuration
│   ├── BACKEND.md          ← cloud API, database, dashboard
│   ├── TESTING.md          ← calibration & accuracy tests
│   └── DEPLOYMENT.md       ← site installation at CLSU SSU
└── references/
    └── safeway.pdf         ← capstone paper (44 pp)
```

## Build Guide

The complete build is organized into five stage guides. Follow them in order:

| Step | Guide | What you'll do |
|---:|---|---|
| 1 | **[docs/HARDWARE.md](docs/HARDWARE.md)** | Wire the laser gates, buzzer, power rails; assemble into the weatherproof enclosure; prep the microSD |
| 2 | **[docs/FIRMWARE.md](docs/FIRMWARE.md)** | Set up Arduino IDE, configure & flash the ESP32-CAM, tune gate distance + speed limit |
| 3 | **[docs/BACKEND.md](docs/BACKEND.md)** | Stand up the cloud API + database + monitoring dashboard |
| 4 | **[docs/TESTING.md](docs/TESTING.md)** | Calibrate gates, verify speed accuracy, run the ISO-based evaluation |
| 5 | **[docs/DEPLOYMENT.md](docs/DEPLOYMENT.md)** | Site survey, pole-mount at the pilot road, train SSU users, maintenance plan |

Each guide is self-contained with wiring tables, commands, and checklists.

## Quick Start (TL;DR)

```bash
git clone https://github.com/qppd/safeway.git
```

1. Order parts from [docs/BOM.md](docs/BOM.md) (~₱1,650 recommended build)
2. Wire per [HARDWARE.md](docs/HARDWARE.md) → flash per [FIRMWARE.md](docs/FIRMWARE.md)
3. Launch the API per [BACKEND.md](docs/BACKEND.md), point the firmware at it
4. Drive/walk through both gates — verify speed + photo + dashboard record

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
| [HARDWARE.md](docs/HARDWARE.md) | Pin map, wiring tables, power design, enclosure & laser alignment |
| [FIRMWARE.md](docs/FIRMWARE.md) | Full annotated firmware sketch, Arduino IDE setup, flashing procedure |
| [BACKEND.md](docs/BACKEND.md) | REST API spec, database schema, dashboard, plate recognition pipeline |
| [TESTING.md](docs/TESTING.md) | Bench tests, speed calibration, evaluation protocol + log templates |
| [DEPLOYMENT.md](docs/DEPLOYMENT.md) | Site survey checklist, outdoor mounting, training, maintenance |

## Team

BS Information Technology · Central Luzon State University

- Roise Anthony M. Barles
- Andrei P. Bernardo
- Marcos A. Salazar

Adviser: Ryan L. Bermoza · Department of Information Technology, College of Engineering

## Acknowledgment

Pilot client: **CLSU Security & Safety Unit**. This project follows the PPDIOO network lifecycle framework (Cisco). The full academic manuscript is in [`references/safeway.pdf`](references/safeway.pdf).

Per the capstone disclaimer: *"The project report or any portion thereof including the source code, or any section may be freely copied and distributed provided that the source is acknowledged."*
