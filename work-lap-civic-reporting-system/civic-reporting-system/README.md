# Civic Issue Reporting System

**For a Better City** — A location-aware civic issue reporting tool for Bangalore city government.

> **Full project documentation:** See [DOCUMENTATION.md](./DOCUMENTATION.md) for architecture, API contracts, RBAC, design decisions, and data model.

## Quick Start

```bash
docker compose up --build
```

(If you have the legacy standalone CLI, use `docker-compose` instead.)

| Service    | URL                     |
|-----------|-------------------------|
| Frontend  | http://localhost:3000   |
| Service A | http://localhost:4000   |
| Service B | http://localhost:5000   |

**Public:** Submit issues at http://localhost:3000  
**Internal:** Government dashboard at http://localhost:3000/internal

---

## Architecture

```
┌─────────────┐     Geolocation API      ┌─────────────┐
│   Browser   │ ◄──────────────────────► │  Frontend   │
│   (User)    │                          │  (Public)   │
└──────┬──────┘                          └──────┬──────┘
       │                                         │
       │ POST /api/v1/tickets                    │
       └────────────────────────────────────────►│
                                                 │
                    ┌────────────────────────────┘
                    ▼
            ┌───────────────┐     Reverse Geocode + City Check
            │   Service A   │ ◄───────────────────────────────► Nominatim (OSM)
            │ Intake/Valid  │
            └───────┬───────┘
                    │ Redis Streams (Pub/Sub)
                    ▼
            ┌───────────────┐
            │   Service B   │     SQLite
            │   Storage     │ ◄────────── tickets.db
            └───────┬───────┘
                    │
                    │ JWT Auth + RBAC
                    ▼
            ┌───────────────┐
            │ Gov Employee  │
            │  Internal UI  │
            └───────────────┘
```

**Flow:**
1. User submits form with lat/lng from Geolocation API.
2. Service A validates input, reverse geocodes via Nominatim, checks Bangalore boundary.
3. Outside city → 403; Inside → publish to Redis Stream.
4. Service B consumes stream, stores in SQLite.
5. Government employees access tickets via internal API/UI with role-based visibility.

---

## API Contracts

### Public API (Service A)

#### `POST /api/v1/tickets`

Submit a civic issue.

**Request:**
```json
{
  "concern": "Pothole",
  "notes": "Large pothole near the junction",
  "userName": "Rahul Kumar",
  "contact": "rahul@email.com",
  "lat": 12.9716,
  "lng": 77.5946
}
```

| Field   | Type   | Required | Description                    |
|---------|--------|----------|--------------------------------|
| concern | string | yes      | Issue type (1–100 chars)       |
| notes   | string | no       | Description (max 2000 chars)   |
| userName| string | yes      | Reporter name (1–200 chars)    |
| contact | string | no       | Phone/email (max 100 chars)    |
| lat     | number | yes      | Latitude (-90 to 90)           |
| lng     | number | yes      | Longitude (-180 to 180)        |

**Success (200):**
```json
{ "ticketId": "550e8400-e29b-41d4-a716-446655440000" }
```

**Outside city (403):**
```json
{
  "error": "Location is outside Bangalore city limits",
  "code": "OUTSIDE_CITY"
}
```

**Validation error (422):**
```json
{
  "error": "Invalid input",
  "code": "VALIDATION_ERROR",
  "details": "concern: Required"
}
```

**Geocoding failure (503):**
```json
{
  "error": "Location validation service temporarily unavailable",
  "code": "GEOCODE_FAILED"
}
```

---

### Internal API (Service B)

#### `POST /internal/login`

Obtain JWT for internal access.

**Request:**
```json
{
  "role": "OFFICER",
  "employeeId": "emp001"
}
```

**Response:**
```json
{
  "token": "eyJhbGciOiJIUzI1NiIs...",
  "role": "OFFICER"
}
```

#### `GET /internal/tickets`

List tickets. Requires `Authorization: Bearer <token>`.

- **Officer:** Limited fields (id, concern, area, timestamp)
- **Supervisor:** Full ticket data

#### `GET /internal/tickets/:id`

Get single ticket. Same RBAC applies.

---

## RBAC Design

| Role       | Permission                              | Ticket visibility                                           |
|------------|-----------------------------------------|-------------------------------------------------------------|
| OFFICER    | View tickets (limited)                  | id, concern, area, timestamp only                           |
| SUPERVISOR | View all tickets (full)                 | Full row: id, concern, notes, userName, contact, lat, lng, area, timestamp |

**Rules:**
- RBAC enforced server-side; no client trust.
- JWT signed with `JWT_SECRET`; role in payload.
- Invalid/missing token → 401; valid token but insufficient role → 403.

---

## Data Model

**tickets**

| Column   | Type    | Description                    |
|----------|---------|--------------------------------|
| id       | TEXT PK | UUID                          |
| concern  | TEXT    | Issue type                    |
| notes    | TEXT    | Description                   |
| userName | TEXT    | Reporter name                 |
| contact  | TEXT    | Phone/email (optional)        |
| lat      | REAL    | Latitude                      |
| lng      | REAL    | Longitude                     |
| area     | TEXT    | Resolved locality name        |
| timestamp| INTEGER | Unix ms                       |

---

## Design Decisions & Tradeoffs

| Decision            | Rationale                                                                 |
|---------------------|---------------------------------------------------------------------------|
| **Nominatim (OSM)** | Free, no API key, suitable for reverse geocoding and locality names. Rate limits apply; we handle retries and failures. |
| **Redis Streams**   | At-least-once delivery, consumer groups for scaling, durable. Simpler than Kafka for this scale. |
| **SQLite**          | Single-file, no extra process, good for dev and low-volume internal use. |
| **JWT for internal**| Stateless auth; suitable for internal employees. For production, use OAuth2/SSO. |
| **403 for outside** | Indicates “forbidden” (outside jurisdiction). 400 reserved for bad payload. |
| **INSERT OR IGNORE**| Idempotency against duplicate messages from stream replay.                |

### Third-Party API (Nominatim)

- **Why:** Free, open data, no API key; sufficient for reverse geocoding.
- **Handling:** 5s timeout, 3 retries with backoff; 503 returned on persistent failure. User-Agent set per usage policy.

### Pub/Sub (Redis Streams)

- **Delivery:** At-least-once; consumer tracks last processed ID.
- **Failure:** Consumer retries on error; `INSERT OR IGNORE` avoids duplicate tickets on replay.

---

## Environment

| Variable    | Service   | Description                     |
|-------------|-----------|---------------------------------|
| REDIS_URL   | A, B      | Redis connection (e.g. `redis://redis:6379`) |
| JWT_SECRET  | B         | Secret for signing JWTs        |

**Note:** Set `JWT_SECRET` via env in production; avoid hardcoding.

---

## Manual JWT (Alternative to /internal/login)

For scripts or testing:

```bash
# Officer
node -e "console.log(require('jsonwebtoken').sign({role:'OFFICER'}, 'supersecret', {expiresIn:'24h'}))"

# Supervisor
node -e "console.log(require('jsonwebtoken').sign({role:'SUPERVISOR'}, 'supersecret', {expiresIn:'24h'}))"
```

Use: `Authorization: Bearer <token>`

---

## Project Structure

```
civic-reporting-system/
├── docker-compose.yml
├── frontend/             # React (Vite) SPA
│   ├── src/
│   │   ├── pages/        # ReportIssue, InternalDashboard
│   │   └── ...
│   └── index.html        # Vite entry
├── service-a/            # Intake, validation, pub/sub
├── service-b/            # Storage, internal API, RBAC
└── README.md
```
