# Civic Issue Reporting System — Project Documentation

**For a Better City** — A location-aware civic issue reporting tool for Bangalore city government.

---

## 1. High-Level Architecture Diagram / Writeup

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         CIVIC ISSUE REPORTING SYSTEM                              │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                   │
│  ┌──────────────────┐                                                            │
│  │     BROWSER      │  Geolocation API (lat/lng)                                  │
│  │  (Resident)      │◄────────────────────────────► Device GPS / Manual Coords   │
│  └────────┬─────────┘                                                            │
│           │                                                                       │
│           │ POST /api/v1/tickets                                                  │
│           ▼                                                                       │
│  ┌──────────────────┐                                                            │
│  │    FRONTEND      │  Static HTML, Port 3000                                     │
│  │  • React (Vite)   │  Public form (/) + Internal dashboard (/internal)           │
│  └────────┬─────────┘                                                            │
│           │                                                                       │
│           │ HTTP                                                                  │
│           ▼                                                                       │
│  ┌──────────────────┐         ┌─────────────────────┐                            │
│  │    SERVICE A     │────────►│  Nominatim (OSM)    │  Reverse Geocode            │
│  │  Intake/Valid    │  API    │  Third-Party API    │  Resolve area/locality      │
│  │  Port 4000       │◄────────│                     │  Verify city boundary       │
│  └────────┬─────────┘         └─────────────────────┘                            │
│           │                                                                       │
│           │ XADD (Redis Streams)                                                  │
│           ▼                                                                       │
│  ┌──────────────────┐                                                            │
│  │      REDIS       │  Stream: tickets-stream                                     │
│  │  Port 6379       │  Pub/Sub message broker                                     │
│  └────────┬─────────┘                                                            │
│           │                                                                       │
│           │ XREAD (Consumer)                                                      │
│           ▼                                                                       │
│  ┌──────────────────┐         ┌─────────────────────┐                            │
│  │    SERVICE B     │────────►│  SQLite             │  tickets.db                 │
│  │  Storage & API   │  INSERT │  Persistent storage │  ./data/tickets.db          │
│  │  Port 5000       │         └─────────────────────┘                            │
│  └────────┬─────────┘                                                            │
│           │                                                                       │
│           │ GET /internal/tickets (JWT Auth)                                      │
│           ▼                                                                       │
│  ┌──────────────────┐                                                            │
│  │  GOV EMPLOYEES   │  Internal UI (/internal)                                    │
│  │  Officer/Super   │  Role-based ticket visibility                               │
│  └──────────────────┘                                                            │
│                                                                                   │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Architecture Writeup

The system follows a **microservices-style** design with clear separation of concerns:

| Component | Responsibility |
|-----------|----------------|
| **Frontend** | Public form for residents; internal dashboard for government staff. No authentication for public; JWT for internal. |
| **Service A** | Stateless intake API. Validates input, calls Nominatim for reverse geocoding and city verification, publishes enriched tickets to Redis Streams. Returns 403 if outside city; 200 + publishes if inside. |
| **Redis** | Message broker (Redis Streams). Decouples intake from storage. At-least-once delivery. |
| **Service B** | Background consumer + HTTP API. Consumes stream, stores in SQLite. Serves internal API with JWT + RBAC. |
| **Nominatim** | Third-party OpenStreetMap service. Reverse geocodes lat/lng to address; used for area name and city boundary check. |

**Data flow:** Resident submits → Service A validates + geocodes → Publishes to Redis → Service B consumes → Stores in SQLite → Gov employees query via internal API with role-based visibility.

---

## 2. API Contracts with Example Requests and Responses

### Public API (Service A — Port 4000)

#### `POST /api/v1/tickets`

Submit a civic issue. No authentication required.

**Example Request:**

```http
POST http://localhost:4000/api/v1/tickets
Content-Type: application/json

{
  "concern": "Pothole",
  "notes": "Large pothole near MG Road junction, causing traffic issues",
  "userName": "Rahul Kumar",
  "contact": "rahul@email.com",
  "lat": 12.9716,
  "lng": 77.5946
}
```

**Example Response — Success (200 OK):**

```json
{
  "ticketId": "550e8400-e29b-41d4-a716-446655440000"
}
```

**Example Response — Outside City (403 Forbidden):**

```json
{
  "error": "Location is outside Bangalore city limits",
  "code": "OUTSIDE_CITY"
}
```

**Example Response — Validation Error (422 Unprocessable Entity):**

```json
{
  "error": "Invalid input",
  "code": "VALIDATION_ERROR",
  "details": "concern: Required"
}
```

**Example Response — Geocoding Failed (503 Service Unavailable):**

```json
{
  "error": "Location validation service temporarily unavailable",
  "code": "GEOCODE_FAILED"
}
```

**Example Response — Publish Failed (503):**

```json
{
  "error": "Ticket created but delivery failed. Please try again.",
  "code": "PUBLISH_FAILED"
}
```

| Field   | Type   | Required | Constraints        | Description                    |
|---------|--------|----------|--------------------|--------------------------------|
| concern | string | yes      | 1–100 chars        | Issue type                     |
| notes   | string | no       | max 2000 chars     | Description                    |
| userName| string | yes      | 1–200 chars        | Reporter name                  |
| contact | string | no       | max 100 chars      | Phone or email                 |
| lat     | number | yes      | -90 to 90          | Latitude                       |
| lng     | number | yes      | -180 to 180        | Longitude                      |

---

### Internal API (Service B — Port 5000)

#### `POST /internal/login`

Obtain JWT for internal access. No prior authentication.

**Example Request:**

```http
POST http://localhost:5000/internal/login
Content-Type: application/json

{
  "role": "SUPERVISOR",
  "employeeId": "emp001"
}
```

**Example Response — Success (200 OK):**

```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJyb2xlIjoiU1VQRVJWSVNPUiIsImVtcGxveWVlSWQiOiJlbXAwMDEiLCJzdWIiOiJlbXBsb3llZSIsImlhdCI6MTcwNjgxMjAwMCwiZXhwIjoxNzA2ODk4NDAwfQ.xxxxx",
  "role": "SUPERVISOR"
}
```

**Example Response — Invalid Role (400 Bad Request):**

```json
{
  "error": "Valid role (OFFICER or SUPERVISOR) required"
}
```

---

#### `GET /internal/tickets`

List all tickets. Requires `Authorization: Bearer <token>`.

**Example Request:**

```http
GET http://localhost:5000/internal/tickets
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**Example Response — Supervisor (200 OK):** Full ticket data

```json
[
  {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "concern": "Pothole",
    "notes": "Large pothole near MG Road",
    "userName": "Rahul Kumar",
    "contact": "rahul@email.com",
    "lat": 12.9716,
    "lng": 77.5946,
    "area": "MG Road",
    "timestamp": 1706812345678
  }
]
```

**Example Response — Officer (200 OK):** Limited fields

```json
[
  {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "concern": "Pothole",
    "area": "MG Road",
    "timestamp": 1706812345678
  }
]
```

**Example Response — Missing Token (401 Unauthorized):**

```json
{
  "error": "Missing authorization token"
}
```

**Example Response — Invalid Token (401):**

```json
{
  "error": "Invalid or expired token"
}
```

---

#### `GET /internal/tickets/:id`

Get a single ticket by ID. Requires `Authorization: Bearer <token>`.

**Example Request:**

```http
GET http://localhost:5000/internal/tickets/550e8400-e29b-41d4-a716-446655440000
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**Example Response — Supervisor (200 OK):**

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "concern": "Pothole",
  "notes": "Large pothole near MG Road",
  "userName": "Rahul Kumar",
  "contact": "rahul@email.com",
  "lat": 12.9716,
  "lng": 77.5946,
  "area": "MG Road",
  "timestamp": 1706812345678
}
```

**Example Response — Officer (200 OK):** Limited fields

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "concern": "Pothole",
  "area": "MG Road",
  "timestamp": 1706812345678
}
```

**Example Response — Not Found (404):**

```json
{
  "error": "Ticket not found"
}
```

---

## 3. RBAC Design and Rules

### Roles

| Role       | Description                           |
|------------|---------------------------------------|
| **OFFICER**   | Field officer; limited ticket visibility |
| **SUPERVISOR**| Supervisor; full ticket visibility      |

### Permissions Matrix

| Action              | OFFICER | SUPERVISOR |
|---------------------|---------|------------|
| Login               | ✓       | ✓          |
| List tickets        | ✓ (limited) | ✓ (full) |
| Get ticket by ID    | ✓ (limited) | ✓ (full) |

### Ticket Field Visibility

| Field     | OFFICER | SUPERVISOR |
|-----------|---------|------------|
| id        | ✓       | ✓          |
| concern   | ✓       | ✓          |
| area      | ✓       | ✓          |
| timestamp | ✓       | ✓          |
| notes     | ✗       | ✓          |
| userName  | ✗       | ✓          |
| contact   | ✗       | ✓          |
| lat       | ✗       | ✓          |
| lng       | ✗       | ✓          |

### RBAC Rules

1. **Server-side enforcement:** RBAC is enforced in Service B. The client cannot override visibility.
2. **JWT payload:** Role is stored in the JWT payload; token is signed with `JWT_SECRET`.
3. **401 Unauthorized:** Missing token, invalid token, or expired token.
4. **403 Forbidden:** Valid token but invalid/unknown role.
5. **No role escalation:** The login endpoint accepts only `OFFICER` or `SUPERVISOR`; no privilege elevation.

---

## 4. Explanation of Design Decisions and Tradeoffs

| Decision | Rationale | Tradeoff |
|----------|-----------|----------|
| **Nominatim (OSM)** | Free, no API key, good for reverse geocoding and area names. | Rate limits; less accurate than paid providers. Mitigated with retries and 503 on failure. |
| **Redis Streams** | Lightweight pub/sub, at-least-once delivery, durable. Simpler than Kafka for this scale. | Single Redis instance; for high scale, consider clustering or Kafka. |
| **SQLite** | Single-file, no separate DB process, easy dev and low-volume deployments. | Single-writer; not ideal for very high concurrency. PostgreSQL for production scale. |
| **JWT for internal auth** | Stateless; no session store. Simple for demo. | For production, prefer OAuth2/SSO with refresh tokens. |
| **403 for outside city** | Semantically “forbidden” (outside jurisdiction). 400 reserved for bad payload. | Clear distinction between validation errors (422) and jurisdiction (403). |
| **INSERT OR IGNORE** | Idempotency on duplicate message replay from stream. | Relies on UUID primary key; duplicates are silently ignored. |
| **Stateless Service A** | Easy horizontal scaling, no session storage. | Each request is independent; no request affinity needed. |
| **Decoupled Pub/Sub** | Service A and B scale independently; intake and storage isolated. | Eventual consistency; ticket may not appear in DB immediately after 200. |
| **Plain HTML frontend** | No build step, quick to run, works with any hosting. | Less structure than a framework; fine for this scope. |

### Third-Party API (Nominatim)

- **Choice:** OpenStreetMap Nominatim.
- **Handling:** 5s timeout, 3 retries with exponential backoff.
- **On failure:** 503 with `GEOCODE_FAILED`; user can retry.

### Pub/Sub (Redis Streams)

- **Delivery:** At-least-once; consumer tracks last processed message ID.
- **Failure:** Consumer retries on error; `INSERT OR IGNORE` prevents duplicate tickets on replay.

---

## 5. README — How to Run or Reason About the System

### Prerequisites

- Docker Desktop (or Docker Engine + Docker Compose)
- Windows / macOS / Linux

### Run the System

```bash
docker compose up --build
```

If using the legacy CLI:

```bash
docker-compose up --build
```

### URLs

| Service   | URL                     |
|-----------|-------------------------|
| Frontend  | http://localhost:3000   |
| Service A | http://localhost:4000   |
| Service B | http://localhost:5000   |

### User Flows

1. **Public submission:** Open http://localhost:3000 → Fill form → Report Issue. Location is validated; if inside Bangalore, ticket is created.
2. **Internal dashboard:** Open http://localhost:3000/internal → Choose Officer or Supervisor → Login → View tickets.

### Stop the System

```bash
docker compose down
```

### Environment Variables

| Variable   | Service   | Description                          |
|------------|-----------|--------------------------------------|
| REDIS_URL  | A, B      | Redis connection (e.g. `redis://redis:6379`) |
| JWT_SECRET | B         | Secret for signing JWTs              |
| DB_PATH    | B         | SQLite file path (default: `tickets.db`) |
| CITY_NAME  | A         | City for boundary check (default: `bangalore`) |

### Project Structure

```
civic-reporting-system/
├── docker-compose.yml
├── frontend/
│   ├── src/
│   │   ├── pages/        # ReportIssue.jsx, InternalDashboard.jsx
│   │   └── ...
│   ├── index.html        # Vite entry
│   ├── package.json
│   ├── vite.config.js
│   └── Dockerfile
├── service-a/            # Intake, validation, pub/sub
│   ├── index.js
│   ├── package.json
│   └── Dockerfile
├── service-b/            # Storage, internal API, RBAC
│   ├── index.js
│   ├── package.json
│   └── Dockerfile
├── data/                 # Persisted SQLite DB (created at runtime)
│   └── tickets.db
├── README.md
└── DOCUMENTATION.md      # This file
```

---

## 6. Data Model

### Table: `tickets`

| Column   | Type    | Constraints | Description                    |
|----------|---------|-------------|--------------------------------|
| id       | TEXT    | PRIMARY KEY | UUID v4                        |
| concern  | TEXT    |             | Issue type (e.g. Pothole, Garbage) |
| notes    | TEXT    |             | Reporter description           |
| userName | TEXT    |             | Reporter name                  |
| contact  | TEXT    | nullable    | Phone or email                 |
| lat      | REAL    |             | Latitude                       |
| lng      | REAL    |             | Longitude                      |
| area     | TEXT    |             | Resolved locality (from Nominatim) |
| timestamp| INTEGER |             | Unix timestamp (ms)            |

### Entity Relationship

Single table; no foreign keys. Each ticket is independent.

### Sample Data

```sql
INSERT INTO tickets VALUES (
  '550e8400-e29b-41d4-a716-446655440000',
  'Pothole',
  'Large pothole near junction',
  'Rahul Kumar',
  'rahul@email.com',
  12.9716,
  77.5946,
  'MG Road',
  1706812345678
);
```
