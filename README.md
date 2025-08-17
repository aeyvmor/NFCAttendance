# NFC UID Attendance System — **ARCHITECTURE.md**

_Raspberry Pi (C++ + LCD) Edge Readers • Node.js + Express API • MongoDB Atlas • React + Material UI • Python Reports (FastAPI) • Monorepo_

> **Final stack (locked):**  
> **Edge**: C++ on Raspberry Pi with NFC reader + small LCD (feedback only)  
> **Backend**: Node.js + Express (REST + SSE/WebSocket)  
> **DB**: MongoDB Atlas (online)  
> **Frontend**: React + Material UI  
> **Reports**: Python microservice (FastAPI + pandas/reportlab)  
> **Repo**: Monorepo (`/device`, `/backend`, `/frontend`, `/reports`, `/deploy`, `/docs`)

---

## Table of Contents

1. [Executive Summary](#executive-summary)  
2. [Goals, Non-Goals, and Constraints](#goals-non-goals-and-constraints)  
3. [Final Technology Choices](#final-technology-choices)  
4. [System Context & Data Flow](#system-context--data-flow)  
5. [Multi-Tenancy, Roles & Auth](#multi-tenancy-roles--auth)  
6. [Data Model (MongoDB)](#data-model-mongodb)  
7. [API Surface (OpenAPI Excerpt)](#api-surface-openapi-excerpt)  
8. [Edge Device (C++ on Raspberry Pi)](#edge-device-c-on-raspberry-pi)  
9. [Backend (Node.js + Express)](#backend-nodejs--express)  
10. [Realtime (SSE / WebSocket)](#realtime-sse--websocket)  
11. [Reports Service (Python + FastAPI)](#reports-service-python--fastapi)  
12. [Frontend (React + Material UI)](#frontend-react--material-ui)  
13. [Operational Flows](#operational-flows)  
14. [Security, Privacy & Compliance](#security-privacy--compliance)  
15. [Environments & Configuration](#environments--configuration)  
16. [Deployment & DevOps](#deployment--devops)  
17. [Observability (Logs, Metrics, Alerts)](#observability-logs-metrics-alerts)  
18. [Testing Strategy](#testing-strategy)  
19. [Monorepo Layout](#monorepo-layout)  
20. [Seed Data (Examples)](#seed-data-examples)  
21. [Appendix: Wireframes & Diagrams](#appendix-wireframes--diagrams)  
22. [Appendix: Example Configs](#appendix-example-configs)

---

## Executive Summary

We are building a **reliable, fast, and low-complexity attendance system** for multiple universities. Students tap NFC cards; the Raspberry Pi edge device (C++) reads the **UID**, sends it to the **Node.js/Express backend**, which validates enrollment and an **active session** and writes to **MongoDB Atlas**. The **React + MUI dashboard** shows live roll. A **Python (FastAPI)** reports service generates CSV/PDF analytics.

**Why UID:** universally readable, no keys/sector access, fewer read failures, simpler hardware/software. UID cloning is theoretically possible; process & system mitigations are in place.

---

## Goals, Non-Goals, and Constraints

### ✅ Goals
- Robust NFC attendance with multi-university tenancy.
- Edge devices provide **instant LCD feedback**: Present/Late/Unknown/No Session.
- Instructor dashboard: Schedules → Sessions (manual start) → Live roll → Reports.
- Online database (MongoDB Atlas) with strict tenant isolation.
- Python reports (analytics + PDF/CSV) for coursework/advanced analysis.

### 🚫 Non-Goals (now)
- Campus SSO (e.g., SAML/OAuth with university IdP).
- LMS gradebook writeback.
- Biometric verification.

### Constraints
- Internet may be unreliable on-site → device offline queue + retry.
- Authoritative time is **server time**; device timestamps are advisory.
- Privacy: store minimal PII (student number, name, section), **no card dumps**.

---

## Final Technology Choices

| Layer | Choice | Rationale |
|---|---|---|
| Edge Device | **C++** on Raspberry Pi + NFC (PC/SC or PN532) + small LCD (I2C/SPI) | Low latency, direct hardware control, minimal footprint. LCD is feedback-only to keep code simple. |
| Backend | **Node.js + Express** | Async IO, easy REST + SSE, perfect with MongoDB, rich ecosystem. |
| Database | **MongoDB Atlas** | Cloud, online, flexible schema; enforce integrity with indexes + Mongoose. |
| Frontend | **React + Material UI** | Modern dashboard, accessible components, fast dev. |
| Reports | **Python + FastAPI** | `pandas` for analytics, `reportlab`/HTML→PDF for exports, course alignment. |
| Auth | **JWT** (access + refresh) | Lightweight stateless auth; per-device API keys for edge. |
| Realtime | **SSE** (simple) or **WebSocket** (optional) | Live table updates during sessions. |
| Packaging | **Docker** (monorepo) | Consistent builds across services. |
| Reverse Proxy | **Caddy** (or Nginx) | TLS, routing `/api`, `/reports`, and static frontend. |

---

## System Context & Data Flow

```mermaid
flowchart LR
  subgraph EDGE["Edge: Raspberry Pi (C++)"]
    RDR[NFC Reader]
    LCD[Small LCD]
    AGT[Edge Agent (C++)]
  end

  subgraph API["Backend: Node.js + Express"]
    AUTH[JWT & RBAC]
    ATT[Attendance Logic]
    SSE[Realtime (SSE)]
  end

  subgraph DB["MongoDB Atlas"]
    U[(universities)]
    US[(users)]
    ST[(students)]
    SC[(student_cards)]
    CO[(courses)]
    EN[(enrollments)]
    SD[(schedules)]
    SE[(sessions)]
    AT[(attendance)]
    DV[(devices)]
    HB[(device_heartbeats)]
    AU[(audit_logs)]
  end

  subgraph UI["React + MUI Dashboard"]
    INS[Instructor UI]
    ADM[Admin UI]
  end

  subgraph RPT["Python Reports (FastAPI)"]
    PDF[PDF/CSV Generation]
    ANA[Analytics (pandas)]
  end

  RDR --> AGT --> LCD
  AGT -- HTTPS+DeviceKey --> API
  API <--> DB
  API --> SSE
  UI <---> API
  UI <---> SSE
  API <---> RPT
  RPT <--> DB
```

**Happy Path:** Tap → Edge reads uidHex → POST to /attendance/checkin → Backend validates session & enrollment → Insert attendance (P/L) → SSE to dashboard → LCD shows result.

---

## Multi-Tenancy, Roles & Auth

**Tenancy:** all records carry `universityId`. Middleware injects `ctx.universityId` from user JWT; device requests include `device.universityId`.

**Roles:** `superadmin` (global), `admin` (university-scoped), `instructor` (course-scoped).

**Auth:**
- **Users:** JWT (short-lived access + refresh rotation). Passwords hashed using Argon2id.
- **Devices:** `X-Device-Key` with allowlist and `active=true`.

**RBAC invariants:**
- Instructors can only access courses they own and derived records.
- Admins can manage users/devices within their `universityId`.
- All write operations log to `audit_logs`.

---

## Data Model (MongoDB)

We prefer references over heavy embedding. All collections indexed by `universityId` where applicable.

### Collections & Key Indexes

- **universities:** `{ _id, name }`
- **users:** `{ _id, universityId, email (unique per uni), passwordHash, role }`
  - Index: `{ universityId:1, email:1 }` unique
- **students:** `{ _id, universityId, studentNumber, fullName, section, archivedAt? }`
  - Index: `{ universityId:1, studentNumber:1 }` unique
- **student_cards:** `{ _id, studentId, uidHex (lowercase), active, createdAt }`
  - Index: `{ uidHex:1 }` unique; partial unique `{ studentId:1, active:1 }` where `active:true`
- **devices:** `{ _id, universityId, identifier, name, location, apiKey, active, lastSeen }`
  - Index: `{ universityId:1, identifier:1 }` unique
- **courses:** `{ _id, universityId, code, name, instructorId }`
  - Index: `{ universityId:1, code:1 }` unique
- **enrollments:** `{ _id, courseId, studentId }`
  - Index: `{ courseId:1, studentId:1 }` unique
- **schedules:** `{ _id, courseId, dayOfWeek(0=Mon..6=Sun), startHHmm, endHHmm, deviceId, graceMinutes }`
- **sessions:** `{ _id, scheduleId?, courseId, date(YYYY-MM-DD), startTime?, endTime?, status('upcoming'|'active'|'closed'|'cancelled') }`
  - Index: `{ courseId:1, date:1 }`, `{ status:1 }`
- **attendance:** `{ _id, sessionId, studentId, checkedInAt, status('P'|'L'), sourceDeviceId }`
  - Index: `{ sessionId:1, studentId:1 }` unique; `{ sessionId:1, checkedInAt:-1 }`
- **device_heartbeats:** `{ _id, deviceId, at, version, commitHash }`
- **audit_logs:** `{ _id, universityId, actorUserId?, action, target, payload, createdAt }`
  - Index: `{ universityId:1, createdAt:-1 }`

### Example Documents

```json
// students
{
  "_id": "64ff...01",
  "universityId": "uni_ust",
  "studentNumber": "23-00123",
  "fullName": "Juan Dela Cruz",
  "section": "MATH165-A",
  "createdAt": "2025-08-01T02:12:00Z"
}
```

```json
// student_cards
{
  "_id": "64ff...02",
  "studentId": "64ff...01",
  "uidHex": "04a1b2c3d4",
  "active": true,
  "createdAt": "2025-08-05T09:00:00Z"
}
```

```json
// sessions
{
  "_id": "sess_124",
  "courseId": "course_math165_ust",
  "date": "2025-08-22",
  "startTime": "2025-08-22T02:02:32Z",
  "endTime": null,
  "status": "active",
  "scheduleId": "sched_ust_math165_mwf_10a",
  "createdAt": "2025-08-22T02:02:32Z"
}
```

```json
// attendance
{
  "_id": "att_9a3",
  "sessionId": "sess_124",
  "studentId": "64ff...01",
  "checkedInAt": "2025-08-22T02:03:12Z",
  "status": "P",
  "sourceDeviceId": "dev_ust_room203"
}
```

---

## API Surface (OpenAPI Excerpt)

Full spec in `/backend/openapi.yml`. This excerpt covers core endpoints.

```yaml
openapi: 3.0.3
info:
  title: NFC UID Attendance API
  version: 1.0.0
servers:
  - url: https://api.example.edu/v1
security:
  - bearerAuth: []
paths:
  /auth/login:
    post:
      summary: User login
      requestBody:
        required: true
        content:
          application/json:
            schema: 
              type: object
              properties:
                email: {type: string}
                password: {type: string}
              required: [email, password]
      responses:
        '200':
          description: JWT tokens
  /devices/register:
    post:
      security: [{ bearerAuth: [] }]
      summary: Admin creates a device; returns API key
  /attendance/checkin:
    post:
      security: []
      summary: Device check-in by UID
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [uidHex, deviceIdentifier]
              properties:
                uidHex: { type: string, example: "04a1b2c3d4" }
                deviceIdentifier: { type: string, example: "pi-ust-room203" }
                capturedAt: { type: string, format: date-time }
      responses:
        '200':
          description: Result of check-in
          content:
            application/json:
              schema:
                type: object
                properties:
                  status: { type: string, enum: [P, L] }
                  student:
                    type: object
                    properties:
                      fullName: { type: string }
                      studentNumber: { type: string }
                  dedup: { type: boolean }
                  unknownUid: { type: boolean }
                  uidLast4: { type: string }
        '404': { description: No active session or unknown device }
  /sessions/start:
    post:
      security: [{ bearerAuth: [] }]
      summary: Start session from a schedule
  /sessions/{id}/end:
    post:
      security: [{ bearerAuth: [] }]
      summary: End session
  /sessions/{id}/attendance:
    get:
      security: [{ bearerAuth: [] }]
      summary: List attendance
  /sessions/{id}/stream:
    get:
      security: [{ bearerAuth: [] }]
    # text/event-stream (SSE)
  /student-cards:
    post:
      security: [{ bearerAuth: [] }]
      summary: Bind UID to student
  /reports/session/{id}/pdf:
    post:
      security: [{ bearerAuth: [] }]
      summary: Generate session PDF (proxy to Reports)
components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
```

---

## Edge Device (C++ on Raspberry Pi)

### Hardware
- **Raspberry Pi 4** (2–4GB), Ethernet preferred (stable latency).
- **NFC Reader:** ACR122U/ACR1252U (PC/SC) or PN532.
- **LCD:**
  - Simple: 16x2 or 20x4 I2C character LCD (HD44780-compatible).
  - Optional Fancy: 1.8"–2.4" SPI TFT (ST7735/ILI9341) later.

### Responsibilities
1. Initialize reader, read UID (lowercase hex).
2. Debounce same UID within 3–5s.
3. POST `/v1/attendance/checkin` with `{ uidHex, deviceIdentifier, capturedAt }`.
4. Handle response → LCD feedback Present/Late/Unknown/No Session.
5. Offline queue: append JSON lines (`/var/lib/nfc-agent/queue.jsonl`) on failure; retry with exponential backoff.
6. Heartbeat: POST `/v1/devices/heartbeat` every 60s.

### State Machine

```mermaid
stateDiagram-v2
  [*] --> Idle
  Idle --> ReadUID: TagDetected
  ReadUID --> Debounce
  Debounce --> PostAPI
  PostAPI --> ShowOK: 200 P/L
  PostAPI --> ShowUnknown: 200 unknownUid
  PostAPI --> QueueOffline: !network OR !2xx
  ShowOK --> Idle: after 1500ms
  ShowUnknown --> Idle: after 2000ms
  QueueOffline --> Idle
```

### Example POST Payload

```json
{
  "uidHex": "04a1b2c3d4",
  "deviceIdentifier": "pi-ust-room203",
  "capturedAt": "2025-08-22T02:03:12Z"
}
```

### Provisioning
1. Admin creates device in dashboard → obtains `identifier` & `apiKey`.
2. Write `/etc/nfc-agent.conf` (root:root, chmod 600).
3. Enable systemd service `nfc-agent.service`.

#### `/etc/nfc-agent.conf`

```ini
SERVER_URL=https://api.example.edu/v1
DEVICE_IDENTIFIER=pi-ust-room203
DEVICE_API_KEY=dev_abc123
DEBOUNCE_MS=3000
LCD_MODE=i2c
```

#### `nfc-agent.service`

```ini
[Unit]
Description=NFC Edge Agent
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=/usr/local/bin/nfc-agent --config /etc/nfc-agent.conf
Restart=always
RestartSec=2
User=root

[Install]
WantedBy=multi-user.target
```

---

## Backend (Node.js + Express)

### Structure

```bash
/backend/src
  /routes
  /controllers
  /models        # Mongoose
  /middlewares
  /services
  /realtime      # SSE helpers
  /utils
```

### Core Logic
- **Attendance Status:** compute using server time vs session start + `graceMinutes`.
- **Idempotency:** attendance unique `(sessionId, studentId)`; retap returns the same record.
- **Unknown UID:** respond `{ unknownUid:true, uidLast4 }` and emit SSE `unknown_uid.detected`.
- **RBAC/Tenancy:** middleware asserts `req.user.universityId` or `req.device.universityId` on every query.
- **Audit:** log registration, overrides, exports.

### Pseudocode (Attendance Check-in)

```javascript
// controllers/attendance.js
async function checkin(req, res) {
  const { uidHex, deviceIdentifier, capturedAt } = req.body;
  const device = await Devices.findOne({ identifier: deviceIdentifier, active: true });
  if (!device) return res.status(404).json({ error: 'device_not_found' });

  const universityId = device.universityId;

  const card = await StudentCards.findOne({ uidHex });
  if (!card) {
    sse.emitToUniversity(universityId, 'unknown_uid.detected', { uidLast4: uidHex.slice(-4) });
    return res.json({ unknownUid: true, uidLast4: uidHex.slice(-4) });
  }

  const student = await Students.findOne({ _id: card.studentId, universityId });
  if (!student) return res.status(404).json({ error: 'student_not_found' });

  const activeSession = await Sessions.findOne({
    courseId: { $in: await Courses.idsForEnrolledStudent(student._id) },
    status: 'active'
  });

  if (!activeSession) return res.status(404).json({ error: 'no_active_session' });

  const now = new Date(); // server time
  const start = new Date(activeSession.startTime);
  const graceMin = await resolveGraceMinutes(activeSession);
  const status = (now <= new Date(start.getTime() + graceMin * 60000)) ? 'P' : 'L';

  const existing = await Attendance.findOne({ sessionId: activeSession._id, studentId: student._id });
  if (existing) return res.json({ status: existing.status, student: pickStudent(student), dedup: true });

  const att = await Attendance.create({
    sessionId: activeSession._id,
    studentId: student._id,
    checkedInAt: now,
    status,
    sourceDeviceId: device._id
  });

  sse.emitToSession(activeSession._id, 'attendance.created', {
    student: pickStudent(student),
    status: att.status,
    checkedInAt: att.checkedInAt
  });

  return res.json({ status: att.status, student: pickStudent(student) });
}
```

---

## Realtime (SSE / WebSocket)

### Default: SSE
- `GET /v1/sessions/:id/stream` → text/event-stream
- Events: `attendance.created`, `unknown_uid.detected`, `session.state_changed`.
- Upgrade path: WebSocket if you need instructor → device controls later.

### SSE Event Example

```
event: attendance.created
data: {"student":{"fullName":"Juan Dela Cruz","studentNumber":"23-00123"},"status":"P","checkedInAt":"2025-08-22T02:03:12Z"}
```

---

## Reports Service (Python + FastAPI)

### Responsibilities
- Compute per-session rolls, per-course summaries, late counts, attendance %.
- Generate CSV & PDF (ReportLab or HTML→PDF with WeasyPrint/Prince).
- Expose internal endpoints secured by shared token/mTLS, called by backend.

### Minimal API (internal)
- `POST /reports/session/{id}/pdf` → bytes (application/pdf)
- `GET /reports/course/{id}/summary.csv` → CSV

### Data Access
- Read-only MongoDB user.
- Cache recent report data (e.g., Redis optional) to speed repeated downloads.

---

## Frontend (React + Material UI)

### Pages
- **Home / Today:** cards to start sessions quickly.
- **Schedules:** weekly recurrence editor (device binding, grace).
- **Sessions:** list upcoming/active/closed; create manual sessions.
- **Live Session:** realtime table via SSE; manual override (with reason).
- **Students & Cards:** search, register card flow.
- **Reports:** download CSV/PDF.
- **Admin:** users, devices, imports.

### UX / A11y
- Badges P/L with color and label.
- Mask UIDs in UI (e.g., `****-a1b2`).
- Manila time display (Asia/Manila), store UTC.

---

## Operational Flows

### Check-In (Happy Path)

```mermaid
sequenceDiagram
  participant S as Student
  participant D as Device (C++)
  participant A as API (Express)
  participant M as MongoDB
  participant U as UI (React)
  S->>D: Tap card (UID)
  D->>A: POST /attendance/checkin {uidHex, deviceIdentifier}
  A->>M: Resolve uidHex→student + active session
  M-->>A: OK
  A->>M: Insert attendance (P/L)
  A-->>D: 200 {status, student}
  A-->>U: SSE attendance.created
  D->>D: LCD "Present" or "Late"
```

### Unknown UID → Registration

```mermaid
sequenceDiagram
  D->>A: /attendance/checkin (unknown uid)
  A-->>U: SSE unknown_uid.detected (uidLast4)
  U->>Instructor: Prompt "Register this card?"
  U->>A: POST /student-cards { uidHex, studentNumber }
  A->>M: Bind card→student (deactivate old active card if any)
  A-->>U: 201 Created
```

### Offline Queue (Device)

```mermaid
sequenceDiagram
  D->>D: Network down -> write to /queue.jsonl
  D->>A: Retry with backoff (2s, 4s, 8s… 60s cap)
  A-->>D: 200 OK
  A-->>U: SSE attendance.created (backfilled)
```

### Reports

```mermaid
sequenceDiagram
  U->>A: Generate Session PDF
  A->>R: POST /reports/session/:id/pdf (internal)
  R->>M: Read session + attendance
  R-->>A: PDF bytes
  A-->>U: application/pdf (download)
```

---

## Security, Privacy & Compliance

- **Transport:** HTTPS everywhere; TLS at proxy.
- **Device Auth:** `X-Device-Key` per device; rotate keys; allowlist active devices.
- **User Auth:** JWT (access short, refresh rotation). Argon2id hash, salted.
- **RBAC/Tenancy:** enforce `universityId` on all queries.
- **PII:** minimal: student number, name, section. No card memory dumps; UID only.

### UID cloning risk: mitigate with:
- Proctor visual checks during tap.
- Anomaly detection: same UID across rooms in ±60s → flag.
- (Optional) random in-class PIN prompt for suspicious taps.

### Audit:
- All registration, overrides, exports → `audit_logs`.

### Retention:
- **Attendance:** term + policy (e.g., 5 years).
- **Audit logs:** ≥ 1 year.
- **Heartbeats:** 90 days.

---

## Environments & Configuration

| Key | Description |
|---|---|
| MONGO_URI | MongoDB Atlas URI |
| JWT_SECRET, JWT_EXPIRES, JWT_REFRESH_EXPIRES | Auth secrets |
| REPORTS_BASE_URL, REPORTS_TOKEN | Internal Python service |
| CORS_ORIGIN | Frontend origin |
| TZ | UTC (backend) — frontend formats to Asia/Manila |
| Device conf | `/etc/nfc-agent.conf` (SERVER_URL, DEVICE_IDENTIFIER, DEVICE_API_KEY, DEBOUNCE_MS) |

---

## Deployment & DevOps

### Docker Compose (dev)
- Services: `api`, `frontend`, `reports`, `mongo` (local dev), `proxy` (Caddy).
- Bind mounts for fast reload.

### Production
- **MongoDB Atlas** (managed).
- **Proxy** (Caddy/Nginx) terminates TLS, routes:
  - `/api` → Express
  - `/reports` → FastAPI
  - `/` → Frontend static (or CDN)
- **Backups:** enable Atlas backups (daily + PITR if available).
- **Secrets:** pass via env/secret manager; never commit.

### Example `deploy/docker-compose.dev.yml` (excerpt)

```yaml
version: "3.9"
services:
  api:
    build: ../backend
    env_file: ../backend/.env
    ports: ["8080:8080"]
    depends_on: [mongo]
  reports:
    build: ../reports
    env_file: ../reports/.env
    ports: ["9000:9000"]
    depends_on: [mongo]
  frontend:
    build: ../frontend
    environment:
      - VITE_API_URL=http://localhost:8080/v1
    ports: ["5173:5173"]
  mongo:
    image: mongo:7
    ports: ["27017:27017"]
    volumes: ["mongo_data:/data/db"]
  proxy:
    image: caddy:2
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
    ports: ["80:80", "443:443"]
volumes:
  mongo_data:
```

---

## Observability (Logs, Metrics, Alerts)

### Logs
- Structured JSON (pino/winston) with `ts`, `lvl`, `requestId`, `actor`, `universityId`, `deviceId`.

### Metrics
- check-ins/min
- dedupe rate (% duplicates)
- device heartbeats (lastSeen)
- report latency (p95)
- API error rate (5xx)

### Alerts
- Device silent > 10 min during scheduled class.
- API 5xx > 2% in 5-min window.
- Mongo connection failures.

---

## Testing Strategy

- **Unit:** attendance status calc, RBAC guards, validator schemas.
- **Integration:** device → API → DB → SSE (mock device).
- **E2E:** Cypress or Playwright (login, start session, simulate taps).
- **Load:** tap bursts (e.g., 10 r/s for 60s).
- **UAT:** per university — import students, register cards, live session, export reports.

### Test Data
- 1k students, 3 courses, 14 sessions/course, randomized P/L/absent.

---

## Monorepo Layout

```bash
/attendance-system
  /backend            # Node.js + Express (API + SSE)
    /src
      /routes /controllers /models /services /middlewares /realtime /utils
    package.json  tsconfig.json?  openapi.yml
  /frontend           # React + MUI
    /src
    package.json  vite.config.ts
  /reports            # Python FastAPI reports service
    /app
    requirements.txt
  /device             # C++ edge agent
    /src
    CMakeLists.txt
  /deploy             # docker-compose, Caddyfile, k8s (optional)
  /docs               # this ARCHITECTURE.md, diagrams
  /scripts            # seeds, migrations, utilities
```

---

## Seed Data (Examples)

### `/scripts/seed.universities.json`

```json
[
  { "_id": "uni_ust", "name": "University of Santo Tomas" },
  { "_id": "uni_mapua", "name": "Mapúa University" },
  { "_id": "uni_upd", "name": "UP Diliman" }
]
```

### `/scripts/seed.users.json`

```json
[
  { "universityId": "uni_ust", "email": "admin@ust.edu", "passwordHash": "<argon2>", "role": "admin" },
  { "universityId": "uni_ust", "email": "reyes@ust.edu", "passwordHash": "<argon2>", "role": "instructor" }
]
```

### `/scripts/seed.courses.json`

```json
[
  { "universityId": "uni_ust", "code": "MATH165", "name": "Advanced Calculus", "instructorId": "INST_REYES_ID" }
]
```

### `/scripts/seed.students.json`

```json
[
  { "universityId": "uni_ust", "studentNumber": "23-00001", "fullName": "Juan Dela Cruz", "section": "A" },
  { "universityId": "uni_ust", "studentNumber": "23-00002", "fullName": "Maria Santos", "section": "A" }
]
```

### Mongo import command

```bash
mongoimport --uri "$MONGO_URI" --collection universities --file scripts/seed.universities.json --jsonArray
mongoimport --uri "$MONGO_URI" --collection users        --file scripts/seed.users.json --jsonArray
mongoimport --uri "$MONGO_URI" --collection courses      --file scripts/seed.courses.json --jsonArray
mongoimport --uri "$MONGO_URI" --collection students     --file scripts/seed.students.json --jsonArray
```

---

## Appendix: Wireframes & Diagrams

### System Overview (Mermaid)

```mermaid
graph LR
A[NFC Tap]-->B[Pi (C++)]
B-->C[Backend API]
C-->D[(MongoDB Atlas)]
C-->E[SSE]
E-->F[Instructor Dashboard]
C-->G[Reports Service (Python)]
```

### Live Session (ASCII)

```
MATH165 • Session #124 • Started 10:02 • Grace 10 min • [End Session]
---------------------------------------------------------------------
# | Student # | Name             | Time     | Status | Notes [Edit]
1 | 23-00001  | Juan Dela Cruz   | 10:03:12 |   P    |
2 | 23-00002  | Maria Santos     | 10:13:40 |   L    |
- Unknown card ****-a1b2 -> [Register]
---------------------------------------------------------------------
Present: 28  Late: 3  Not yet: 12
```

---

## Appendix: Example Configs

### `/backend/.env.example`

```ini
NODE_ENV=development
PORT=8080
MONGO_URI=mongodb+srv://user:pass@cluster/attendance?retryWrites=true&w=majority
JWT_SECRET=change_me
JWT_EXPIRES=15m
JWT_REFRESH_EXPIRES=7d
REPORTS_BASE_URL=http://reports:9000
REPORTS_TOKEN=shared_secret
CORS_ORIGIN=http://localhost:5173
```

### `/reports/.env.example`

```ini
MONGO_URI=mongodb+srv://readonly:pass@cluster/attendance
REPORTS_PORT=9000
REPORTS_TOKEN=shared_secret
```

### `/deploy/Caddyfile`

```
attendance.example.edu {
  encode gzip
  tls you@example.com

  @api path /v1* /api*
  reverse_proxy @api api:8080

  @reports path /reports*
  reverse_proxy @reports reports:9000

  route {
    file_server / /srv/frontend
    try_files {path} /index.html
  }
}
```

### HTTP Error Codes (Common)

| Code | Meaning | Notes |
|---|---|---|
| 200 | OK | Successful check-in/export |
| 201 | Created | Registration actions |
| 400 | Bad Request | Validation error |
| 401 | Unauthorized | Missing/invalid JWT |
| 403 | Forbidden | RBAC violation |
| 404 | Not Found | No active session / Unknown device |
| 409 | Conflict | Duplicate attendance / session state |
| 429 | Too Many Requests | Device/IP throttled |
| 500 | Internal Server Error | Unexpected issues |

---

## Final Notes

- **Manual session start** is intentional to avoid ghost sessions and to align with classroom logistics.
- **Absent** is derived at report time (no row written), simplifying writes and idempotency.
- Keep the LCD simple — feedback only; all complex logic remains server-side.
- This document is intended to be committed as `docs/ARCHITECTURE.md`.