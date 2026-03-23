# Access Control System — Project Showcase

> **Commercial software.** Source code is proprietary. No live demo available — this is a self-hosted, on-premises server running exclusively on customer hardware.

This repository is a **project showcase**: a technical overview, architecture description, and visual walkthrough of the access control platform prototype.

---

## Table of Contents

- [Development Approach](#development-approach)
- [Overview](#overview)
- [Key Features](#key-features)
- [Tech Stack](#tech-stack)
- [Architecture](#architecture)
- [Device Integrations](#device-integrations)
- [Security](#security)
- [Screenshots](#screenshots)
- [Licensing Model](#licensing-model)

---
## Development Approach

This project was built entirely through **AI-assisted, specification-driven development**.

Every module, route, adapter, migration script, and UI component was produced by an AI coding agent working from detailed task descriptions. The role of the project owner was to define requirements, describe desired behavior, and verify correctness against real hardware and business logic — without writing or manually editing a single line of code.

---
## Overview

**S-ACCESS** is an autonomous, self-hosted Access Control System (ACS) for Windows environments. It serves as the unified backend for biometric terminals, card readers, and turnstiles — consolidating real-time event streaming, worktime accounting, user management, and device synchronization into a single local server.

**Design goal:** "Set and forget" — the system is architected to operate without maintenance after initial deployment.

**Target:** Mid-to-large organizations (500+ employees) with multiple entry/exit points, shift-based work schedules, and physical security requirements.

---

## Key Features

| Feature | Description |
|---|---|
| **Real-time Monitoring** | Live display of active sessions and access events across all connected devices |
| **User Management** | Full CRUD with photo storage, card assignment, biometric enrollment |
| **Worktime Tracking** | Entry/exit session pairing with duration calculation and shift-aware logic |
| **Multi-Device Sync** | Propagates user data (card numbers, face images) across all physical terminals |
| **Network Auto-Discovery** | Scans local network, detects Hikvision/Dahua devices automatically |
| **Telegram Shift Reports** | Automated end-of-shift summaries sent to a configured Telegram bot |
| **Excel Export** | Download event logs and worktime reports as `.xlsx` |
| **Statistics Dashboard** | Daily/weekly/monthly charts — entry counts, peak hours, access method breakdown |
| **Auto-Backup** | 6-hourly incremental backups with smart rotation (7 daily / 4 weekly / 12 monthly) |
| **Self-Healing Boot** | Integrity check on startup with automatic database recovery from backup |
| **Comment Annotations** | Operators can attach notes to any active session |
| **Offline Operation** | Zero dependency on cloud or internet after activation |

---

## Tech Stack

| Layer | Technology |
|---|---|
| **Runtime** | Node.js 18+ |
| **HTTP Framework** | Express.js 4.18 |
| **Database** | SQLite 3 (WAL mode, crash-safe) |
| **Frontend** | Vanilla JavaScript, HTML5, CSS3 — no frameworks |
| **Password Security** | bcrypt (rounds: 10) |
| **Data Encryption** | AES-256-CBC with per-record IVs |
| **Device Auth** | HTTP Digest (Hikvision) · TCP DVRIP binary protocol (Dahua) |
| **Logging** | Rotating file log (50 MB cap, 5 archives) |
| **Build / Distribution** | `pkg` — compiles to standalone Windows x64 `.exe` |
| **Notifications** | node-telegram-bot-api |
| **Export** | ExcelJS |
| **Parsing** | xml2js (Hikvision XML event stream) |

---

## Architecture

```
┌─────────────────────── S-ACCESS SERVER ──────────────────────────┐
│                                                                  │
│  Express HTTP API  ←→  Session Auth  ←→  SQLite (WAL)            │
│         │                                     │                  │
│   REST Routes                          In-Memory Indexes         │
│   (users/devices/                      uidIndex / cardIndex      │
│    logs/stats/system)                  O(1) lookups              │
│         │                                                        │
│   Service Layer                                                  │
│   ├── eventHandler      (ENTRY/EXIT resolution + sessions)       │
│   ├── deviceManager     (lifecycle, heartbeat, orchestration)    │
│   ├── syncService       (user → device propagation)              │
│   ├── eventSyncService  (historical recovery from devices)       │
│   ├── backupService     (scheduled + manual backups)             │
│   ├── maintenanceService(disk monitor, VACUUM, photo cleanup)    │
│   ├── selfHealService   (boot-time DB integrity + recovery)      │
│   └── shiftReportService(Telegram notifications)                 │
│         │                                                        │
│   Adapter Layer   [Adapter Pattern — abstracts protocols]        │
│   ├── HikvisionAdapter  → HTTP ISAPI / Digest Auth               │
│   └── DahuaAdapter      → TCP DVRIP / Binary protocol            │
│                                                                  │
└──────────────┬─────────────────────────┬─────────────────────────┘
               │                         │
   ┌───────────▼──────────┐  ┌───────────▼──────────┐
   │  Hikvision Terminals │  │    Dahua Turnstiles  │
   │  DS-K1T343M          │  │    ASG-ASI2101       │
   │  Face + Card         │  │    Card              │
   │  Port 80 · REST/XML  │  │    Port 37777 · TCP  │
   └──────────────────────┘  └──────────────────────┘
```

### Key Architecture Decisions

- **No frontend framework** — Vanilla JS ensures decades of compatibility without dependency rot
- **Adapter pattern** — New device vendors can be added without touching business logic
- **Event bus (pub/sub)** — Decoupled event flow between device monitors and session handler
- **Config mirroring** — Settings stored in both SQLite and a JSON sidecar; survives DB corruption
- **Hardware-bound license** — Derived from motherboard UUID + CPU ID; cannot be cloned

---

## Device Integrations

### Hikvision DS-K1T343M — Face Recognition Terminal

- **Protocol:** HTTP REST (ISAPI) with Digest Authentication
- **Capabilities:** Card reader, face recognition
- **Event delivery:** Long-polling chunked XML stream (`/ISAPI/Event/notification/alertStream`)
- **Card format:** 10-digit zero-padded decimal

### Dahua ASG-ASI2101 — Turnstile Controller

- **Protocol:** TCP DVRIP — proprietary binary protocol (32-byte header + JSON payload)
- **Capabilities:** Card reader
- **Event delivery:** TCP subscription with persistent session
- **Card format:** Hexadecimal (e.g. `060BE834`)

### Auto-Discovery

The server probes the local network and identifies device vendors automatically:
- TCP SYN to port `37777` → Dahua
- HTTP probe to port `80` / `8000` → Hikvision

Card UIDs are normalized between formats transparently on both ingest and sync.

---

## Security

| Concern | Implementation |
|---|---|
| Admin password | bcrypt-hashed, never stored in plain text |
| Device passwords | AES-256-CBC encrypted at rest, unique IV per record |
| Biometric photos | AES-256-CBC encrypted at rest (`{uid}.enc`), decrypted only on sync |
| Session tokens | UUID v4, validated on every authenticated request |
| Input validation | Sanitization and type-checking on all API boundaries |
| Timing-safe comparison | `crypto.timingSafeEqual` used for license verification |
| No secrets in logs | Passwords, tokens, and keys never appear in log output |
| Hardware licence binding | License key = SHA-256(HWID + salt); non-transferable |

---

## Screenshots

### 01 — Login Screen
**Preview:** [Login Screen](screenshots/01_login.png)

The entry point to the system. A minimal, focused login form — no registration, no guest access. Authentication uses a single admin password hashed with bcrypt on the server side. 

---

### 02 — Real-Time Monitoring
**Preview:** [Real-Time Monitoring](screenshots/02_monitoring.png)

The main operational view showing all **currently active sessions** — visitors who have entered and not yet exited. Each row displays: name, access method, entry time, elapsed duration and comment section. The list updates automatically without page refresh. At the bottom of the page is a similar table for recent **completed** events.

---

### 03 — Access Event History (Log)
**Preview:** [Access Event History](screenshots/03_history.png)

The full chronological event log with date filtering. Each event shows: timestamp, user(card) name, event type (entry/exit), and access method (card / face/button). Paginated.

---

### 04 — User List
**Preview:** [User List](screenshots/04_users_list.png)

The user directory. Displays UID, name, assigned card numbers and Face ID, user type (guest + subtype / staff). Includes search and filter controls. 

---

### 05 — User Profile / Edit Form
**Preview:** [User Profile](screenshots/05_user_profile.png)

The user detail view with full editing capabilities: personal info, role, access card assignment, biometric photo upload. Card UIDs and Face ID can be assigned by manual entry or by activating scan mode directly from this screen.

---

### 06 — Card / Face Enrollment (Scan Mode)
**Preview:** [Scan Mode](screenshots/06_scanning.png)

A enrollment interface. The operator activates a scan window; the user taps a card or presents a face. The captured data is automatically bound to the target user profile. Biometric photos are encrypted with AES-256 before being written to disk.

---

### 07 — Device Management
**Preview:** [Device Management](screenshots/07_devices.png)

The device control panel lists all connected terminals and turnstiles. Each entry displays the IP address, vendor (Hikvision / Dahua), assigned role (guest / staff) and direction (entry, exit, or bidirectional), and online/offline status. It also allows designating a specific device as the active enrollment station for capturing card UIDs and face images. 

---

### 08 — Network Device Discovery
**Preview:** [Device Discovery](screenshots/08_device_discovery.png)

The auto-discovery panel showing devices found by scanning the local subnet. Vendor type (Hikvision/Dahua) is detected automatically via protocol probing — no manual selection required. The operator reviews the list and adds devices to the system with a single click.

---

### 09 — Statistics Dashboard
**Preview:** [Statistics](screenshots/09_statistics.png)

Charts and metrics for a selected date range: total entries and exits per day, peak access hours, user subtype distribution. The statistical report can be downloaded in .xlsx format.

---

### 10 — Staff Worktime
**Preview:** [Worktime Report](screenshots/10_worktime.png)

Shift Accounting and Employee Status: This view monitors real-time occupancy (Inside / Outside). The system automatically pairs entry and exit events into work sessions for each employee. Users can click on an employee’s name to view a detailed breakdown of all entries and exits per shift.

---

### 11 — Settings Panel
**Preview:** [Settings](screenshots/11_settings.png)

System configuration screen: workday start hour (affects session pairing logic), admin password change. Weekly day off setting prevents zero-traffic days from skewing your reporting accuracy.


---

## Licensing Model

S-ACCESS is distributed as a compiled Windows `.exe` with no external runtime dependencies.

- **License type:** Proprietary, hardware-bound — one license per server machine
- **Activation (BETA):** Each key is derived from the server's motherboard UUID and CPU ID; it cannot be transferred to a different machine
- **Deployment model:** Installed on customer premises by a systems integrator
- **No cloud dependency:** Operates fully offline after activation

---

## Project Status

| Parameter | Value |
|---|---|
| Status | Pre-production (RC1 - Release Candidate) |
| Stage | Beta / Feature-complete |
| Platform | Windows x64 |
| Distribution | Standalone `.exe` compiled via `pkg` |
| Database | SQLite (embedded, no installation required) |
| Supported device vendors | Hikvision, Dahua |
| Min. Node.js (dev environment) | 18+ |
