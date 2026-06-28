# Website-5
Flood Guard Website
[README (4).md](https://github.com/user-attachments/files/29427855/README.4.md)
<div align="center">

<img src="https://img.shields.io/badge/Java-17-ED8B00?style=for-the-badge&logo=openjdk&logoColor=white"/>
<img src="https://img.shields.io/badge/Spring_Boot-3.2-6DB33F?style=for-the-badge&logo=spring&logoColor=white"/>
<img src="https://img.shields.io/badge/React-18-61DAFB?style=for-the-badge&logo=react&logoColor=black"/>
<img src="https://img.shields.io/badge/Kafka-Apache-231F20?style=for-the-badge&logo=apachekafka&logoColor=white"/>
<img src="https://img.shields.io/badge/PostgreSQL-15-316192?style=for-the-badge&logo=postgresql&logoColor=white"/>
<img src="https://img.shields.io/badge/Docker-Compose-2496ED?style=for-the-badge&logo=docker&logoColor=white"/>

# 🛡️ FloodGuard
### Crowdsourced Disaster Management Portal

**A hyper-local, real-time platform for monsoon flood response in Kerala.**  
Connects civilians, field volunteers, and district command through geospatial SOS routing, live volunteer coordination, and resource allocation mapping.

[🚨 Report SOS](#-sos-report-public-page) · [📊 Dashboard](#-operations-dashboard) · [⚡ Quick Start](#-quick-start) · [📡 API Docs](#-rest-api-reference) · [🏗️ Architecture](#-architecture)

</div>

---

## 📋 Table of Contents

- [Overview](#-overview)
- [Live Website](#-live-website)
- [Architecture](#-architecture)
- [Features](#-features)
- [Project Structure](#-project-structure)
- [Quick Start](#-quick-start)
- [Local Development](#-local-development)
- [REST API Reference](#-rest-api-reference)
- [WebSocket Topics](#-websocket-topics-stomp)
- [Environment Variables](#-environment-variables)
- [Tech Stack](#-tech-stack)
- [Database Schema](#-database-schema)
- [Testing](#-testing)
- [Deployment](#-deployment)
- [Use Case](#-use-case)

---

## 🌊 Overview

FloodGuard is a three-layer platform purpose-built for Kerala's monsoon flood seasons:

| Layer | What it does |
|---|---|
| **Public Website** (`floodguard.html`) | Static landing page — live zone status, SOS form, volunteer signup, dashboard preview |
| **React Operations Dashboard** (`/frontend`) | Real-time map, incident management, volunteer coordination, resource tracking |
| **Java Spring Boot API** (`/src`) | JWT-secured REST API + Kafka event bus + STOMP WebSocket broker |

**Key design decision:** Civilians need zero friction to submit SOS — no account, no app install. The public SOS form (`POST /api/incidents`) is the only unauthenticated write endpoint.

---

## 🌐 Live Website

The `floodguard.html` file is a **complete single-file static website** — no build step, no dependencies, opens directly in any browser.

### Sections

| Section | Description |
|---|---|
| **Status Bar** | Sticky live alert strip — "FLOOD WATCH ACTIVE" with real-time IST clock |
| **Hero** | Animated rising floodwater visual + live SOS count, deployed volunteers, evacuated count |
| **Live Stats Strip** | 7 metrics: river level, rainfall, zones, shelters, evacuated, boats, open SOS |
| **How It Works** | Three-step flow: Civilians Report → System Dispatches → Volunteers Respond |
| **Features** | Six platform capabilities with tech badges |
| **Dashboard Preview** | Full browser-chrome mockup with mini map, incident cards, resource bars |
| **SOS Report Form** | GPS-enabled emergency form — real `navigator.geolocation` API, validation, animated submission |
| **Volunteer Roles** | Four roles: Boat Operator, Medical, Logistics Driver, Comms & Coordination |
| **Live Zone Map** | Zone status list + animated CSS map with pulsing SOS pins and rising water |
| **Footer** | Emergency numbers (112, 1077), links, tech stack tags |

### Design tokens

```
Navy:       #0B1F3A   (primary background — command authority)
Alert Red:  #D63B3B   (SOS, critical incidents)
Rescue Amber: #E8941A (high alerts, deployment warnings)
Safe Green: #2D8A4E   (shelters, resources OK, evacuated safe)
Mist:       #EBF2F8   (text on dark)
Blue Accent:#3A8ECC   (map, info, tech tags)
```

**Typefaces:** `Barlow Condensed` (display — urgent, compressed) + `Inter` (body — readable under stress)

### Open the website

```bash
# No build needed — open directly
open floodguard.html

# Or serve locally
npx serve . -p 8000
# → http://localhost:8000/floodguard.html
```

---

## 🏗️ Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                        Public Website                                │
│              floodguard.html  (zero-dependency static)               │
│   Hero · SOS Form · Zone Map · Dashboard Preview · Volunteer CTA     │
└──────────────────────────────────┬───────────────────────────────────┘
                                   │ POST /api/incidents (SOS, public)
┌──────────────────────────────────▼───────────────────────────────────┐
│                    React Operations Dashboard                        │
│                        /frontend  (React 18)                         │
│                                                                      │
│  LiveMap (Leaflet)  ·  Incidents  ·  Volunteers  ·  Resources        │
│  useWebSocket hook (STOMP/SockJS) for real-time push updates         │
│  useAuth hook (JWT context)  ·  Axios service layer                  │
└──────────────────────────────────┬───────────────────────────────────┘
                                   │ REST + WebSocket
┌──────────────────────────────────▼───────────────────────────────────┐
│                   Spring Boot Backend  (Java 17)                     │
│                                                                      │
│  Controllers:  Incident · Volunteer · Resource · Auth · Dashboard    │
│  Services:     IncidentService (auto-dispatch)                       │
│                VolunteerService (GPS tracking)                       │
│                ResourceService (threshold alerts + scheduler)        │
│                JwtService · DashboardService                         │
│  WebSocket:    DisasterWebSocketHandler (STOMP broker)               │
│  Security:     Spring Security + JWT filter (role-based)             │
└──────────────┬───────────────────────────────┬───────────────────────┘
               │                               │
┌──────────────▼──────────┐      ┌─────────────▼──────────────────────┐
│     PostgreSQL 15        │      │           Apache Kafka              │
│     + PostGIS            │      │                                    │
│                          │      │  Topics:                           │
│  Tables:                 │      │   sos-alerts          (3 partitions)│
│   incidents              │      │   resource-updates    (2 partitions)│
│   incident_updates       │      │   volunteer-updates   (2 partitions)│
│   volunteers             │      │   incident-broadcasts (3 partitions)│
│   volunteer_skills       │      │                                    │
│   resources              │      │  Consumers:                        │
│   flood_zones            │      │   floodguard-group                 │
│   users / user_roles     │      └────────────────────────────────────┘
│                          │
│  Spatial queries:        │
│   Haversine distance     │
│   Nearest volunteer      │
│   Radius search          │
└──────────────────────────┘
```

### Request flow — SOS to dispatch

```
Civilian submits SOS (no auth)
  → POST /api/incidents  { isSos: true, requiresBoat: true, lat, lng }
  → IncidentService.createIncident()
  → incidentRepository.save()
  → kafkaTemplate.send("sos-alerts", incident)          ← async broadcast
  → messagingTemplate.convertAndSend("/topic/incidents") ← WebSocket push
  → attemptAutoAssignment()
      → volunteerRepository.findNearestAvailableBoatRescuers(lat, lng, 10km)
      → assignVolunteer(incidentId, bestVolunteer.id)
      → volunteer.status = DEPLOYED
      → incident.status  = ASSIGNED
      → WebSocket push to /topic/incidents
```

---

## 🚀 Features

### 🗺️ Live Geospatial Map
- Leaflet map with OpenStreetMap tiles
- Flood zones as colour-coded circles (PostGIS `radiusKm` → Leaflet `Circle`)
- SOS incident pins, shelter markers, rescue boat positions
- Volunteer GPS positions — updated live via WebSocket every check-in
- Layer switcher: All / Flood Zones / Volunteers only

### 🚨 SOS Auto-Dispatch
- Civilians submit from any device, zero account required
- Immediate Kafka event on `sos-alerts` topic
- Haversine-formula native SQL query finds nearest available volunteer
- Filters: boat-capable volunteers for water rescue, medical-trained for health emergencies
- Auto-assigns and sets volunteer `DEPLOYED`, incident `ASSIGNED`

### 👥 Volunteer Coordination
- Self-registration with skill profile (rescue cert, medical training, boat, vehicle)
- Zone assignment by coordinators
- Live GPS check-in via `PATCH /api/volunteers/{id}/location`
- Status lifecycle: `STANDBY → ACTIVE → DEPLOYED → ACTIVE` (freed on resolution)
- Filter by skill, zone, status, or proximity

### 📦 Resource Allocation
- 8 resource categories: FOOD, WATER, MEDICAL, RESCUE_EQUIPMENT, SHELTER, VEHICLES, POWER, COMMUNICATION
- Per-item `criticalThreshold` and `warningThreshold`
- Auto-alert when stock crosses threshold: WebSocket push + Kafka event
- `@Scheduled` check every 15 minutes — re-broadcasts any critical items
- Zone allocation tracking

### 🔐 Security
- JWT tokens (HS256, 24h expiry)
- Three roles: `ROLE_ADMIN`, `ROLE_COORDINATOR`, `ROLE_FIELD_OFFICER`
- Public endpoints: `POST /api/incidents`, `GET /api/incidents/*`, `/api/auth/**`, `/ws/**`
- `@PreAuthorize` method-level security on sensitive operations

### 📡 Real-Time
- STOMP over SockJS WebSocket
- Four broadcast topics: incidents, volunteer locations, resources, resource alerts
- Clients subscribe with `@stomp/stompjs` + SockJS, auto-reconnect on drop

---

## 🗂️ Project Structure

```
floodguard/
│
├── floodguard.html                    ← Static public website (no build)
│
├── pom.xml                            ← Maven dependencies
├── Dockerfile                         ← Multi-stage Java 17 → JRE alpine
├── docker-compose.yml                 ← Full stack: Postgres + Kafka + Backend + Frontend
│
├── src/
│   ├── main/
│   │   ├── java/com/floodguard/
│   │   │   │
│   │   │   ├── FloodGuardApplication.java        ← Entry point + @EnableScheduling
│   │   │   │
│   │   │   ├── config/
│   │   │   │   ├── SecurityConfig.java           ← JWT filter, RBAC, BCrypt
│   │   │   │   ├── WebSocketConfig.java          ← STOMP broker, SockJS endpoint
│   │   │   │   ├── KafkaConfig.java              ← Topic definitions (4 topics)
│   │   │   │   └── GlobalExceptionHandler.java   ← @RestControllerAdvice
│   │   │   │
│   │   │   ├── controller/
│   │   │   │   ├── IncidentController.java       ← CRUD + assign + nearby
│   │   │   │   ├── VolunteerController.java      ← Register + location + zone
│   │   │   │   ├── ResourceController.java       ← Inventory + quantity + zone
│   │   │   │   ├── AuthController.java           ← Login + register
│   │   │   │   └── DashboardController.java      ← Summary + zone endpoints
│   │   │   │
│   │   │   ├── dto/
│   │   │   │   └── DTOs.java                     ← All request/response DTOs
│   │   │   │
│   │   │   ├── entity/
│   │   │   │   ├── Incident.java                 ← SOS/incident with geo coords
│   │   │   │   ├── IncidentUpdate.java           ← Audit trail per incident
│   │   │   │   ├── Volunteer.java                ← Skills, GPS, status
│   │   │   │   ├── Resource.java                 ← Inventory with thresholds
│   │   │   │   ├── FloodZone.java                ← Zone with severity + water level
│   │   │   │   └── User.java                     ← Ops users with roles
│   │   │   │
│   │   │   ├── enums/
│   │   │   │   ├── Severity.java                 ← CRITICAL, HIGH, MODERATE, LOW, INFO
│   │   │   │   ├── IncidentStatus.java           ← OPEN, ASSIGNED, IN_PROGRESS, RESOLVED, CLOSED
│   │   │   │   ├── VolunteerStatus.java          ← ACTIVE, STANDBY, DEPLOYED, UNAVAILABLE, OFF_DUTY
│   │   │   │   └── ResourceCategory.java         ← FOOD, WATER, MEDICAL, RESCUE_EQUIPMENT…
│   │   │   │
│   │   │   ├── repository/
│   │   │   │   ├── IncidentRepository.java       ← Haversine radius query, SOS filter
│   │   │   │   ├── VolunteerRepository.java      ← Nearest boat rescuer, by skill/zone
│   │   │   │   ├── ResourceRepository.java       ← Critical/low threshold queries
│   │   │   │   ├── FloodZoneRepository.java      ← By severity, evacuation filter
│   │   │   │   └── UserRepository.java
│   │   │   │
│   │   │   ├── service/
│   │   │   │   ├── IncidentService.java          ← Create, assign, auto-dispatch, resolve
│   │   │   │   ├── VolunteerService.java         ← Register, GPS update, status lifecycle
│   │   │   │   ├── ResourceService.java          ← Quantity update, alerts, @Scheduled check
│   │   │   │   ├── JwtService.java               ← Token generate/validate (jjwt)
│   │   │   │   └── DashboardService.java         ← Aggregated summary stats
│   │   │   │
│   │   │   └── websocket/
│   │   │       └── DisasterWebSocketHandler.java ← @MessageMapping: /sos, /volunteer/location
│   │   │
│   │   └── resources/
│   │       ├── application.yml                   ← All config with env var overrides
│   │       └── data.sql                          ← Seed: 2 users, 4 zones, 8 volunteers, 13 resources, 5 incidents
│   │
│   └── test/java/com/floodguard/
│       ├── IncidentServiceTest.java              ← 4 tests: SOS, resolve, assign, not-found
│       └── ResourceServiceTest.java              ← 5 tests: threshold alerts, utilisation %
│
└── frontend/
    ├── package.json
    ├── Dockerfile                                ← Multi-stage React build → nginx
    ├── nginx.conf                                ← SPA routing + API proxy + WS proxy
    └── src/
        ├── App.jsx                               ← Router + ProtectedRoute
        ├── hooks/
        │   ├── useAuth.js                        ← JWT context, login/logout, hasRole()
        │   └── useWebSocket.js                   ← STOMP client, auto-reconnect, publish()
        ├── pages/
        │   ├── Dashboard.jsx                     ← Summary stats
        │   ├── LiveMap.jsx                       ← Leaflet map, real-time pins
        │   ├── Incidents.jsx                     ← Incident list + status management
        │   ├── Volunteers.jsx                    ← Volunteer table + assignment
        │   ├── Resources.jsx                     ← Resource cards + quantity update
        │   ├── Login.jsx                         ← JWT login form
        │   └── SosReport.jsx                     ← Public SOS form (GPS + no auth)
        └── services/
            └── api.js                            ← Axios client: incidentApi, volunteerApi, resourceApi, authApi
```

---

## ⚡ Quick Start

### Prerequisites

| Tool | Version | Install |
|---|---|---|
| Java | 17+ | [adoptium.net](https://adoptium.net) |
| Maven | 3.9+ | Bundled via `./mvnw` |
| Docker | 24+ | [docker.com](https://docker.com) |
| Docker Compose | 2.x | Included with Docker Desktop |
| Node.js | 20+ | [nodejs.org](https://nodejs.org) (frontend dev only) |

### One-command startup

```bash
# 1. Clone
git clone https://github.com/YOUR_USERNAME/floodguard.git
cd floodguard

# 2. Launch full stack
docker-compose up --build
```

| URL | Service |
|---|---|
| `http://localhost:3000` | React operations dashboard |
| `http://localhost:3000/report` | Public civilian SOS form |
| `http://localhost:8080/api` | Spring Boot REST API |
| `http://localhost:8080/api/actuator/health` | Health check |
| `file:///path/to/floodguard.html` | Static public website |

### Default credentials

| Username | Password | Role | Access |
|---|---|---|---|
| `admin` | `admin123` | `ROLE_ADMIN` | Full access |
| `coordinator1` | `coord123` | `ROLE_COORDINATOR` | Incidents, volunteers, resources |

> ⚠️ Change all credentials before any production deployment.

---

## 🔧 Local Development

### Backend only (fastest iteration)

```bash
# Start just the infrastructure
docker-compose up postgres kafka -d

# Run Spring Boot with hot reload
./mvnw spring-boot:run

# API is live at http://localhost:8080/api
```

### Frontend only

```bash
cd frontend
npm install
npm start
# → http://localhost:3000 (proxies /api to :8080)
```

### Static website

```bash
# No build required — open directly in browser
open floodguard.html

# Or serve from any static host
npx serve . -p 4000
```

### Run tests

```bash
# All tests
./mvnw test

# Single class
./mvnw test -Dtest=IncidentServiceTest

# With coverage report
./mvnw test jacoco:report
# → target/site/jacoco/index.html
```

---

## 📡 REST API Reference

All endpoints are prefixed with `/api`.

### Authentication

```http
POST /api/auth/login
Content-Type: application/json

{ "username": "admin", "password": "admin123" }

→ 200 OK
{
  "token": "eyJ...",
  "username": "admin",
  "roles": ["ROLE_ADMIN"],
  "expiresIn": 86400000
}
```

```http
POST /api/auth/register                    [ADMIN only]
{ "username": "officer1", "password": "pass", "email": "...", "role": "FIELD_OFFICER" }
```

---

### Incidents

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| `POST` | `/incidents` | ❌ Public | Create incident / SOS |
| `GET` | `/incidents` | ✅ | List active (OPEN) incidents |
| `GET` | `/incidents/{id}` | ✅ | Get single incident |
| `GET` | `/incidents/sos` | ✅ | Active SOS alerts only |
| `GET` | `/incidents/zone/{zone}` | ✅ | By zone name |
| `GET` | `/incidents/nearby` | ✅ | `?lat=&lng=&radius=` (km) |
| `GET` | `/incidents/stats` | ✅ | Count by status/severity |
| `PATCH` | `/incidents/{id}/status` | 🔑 Coord+ | Update status |
| `POST` | `/incidents/{id}/assign/{volId}` | 🔑 Coord+ | Assign volunteer |

**Create SOS example:**
```http
POST /api/incidents
Content-Type: application/json

{
  "title": "Family of 4 trapped — Ward 7",
  "description": "Ground floor submerged. Elderly member needs medical help.",
  "severity": "CRITICAL",
  "latitude": 10.7872,
  "longitude": 76.3865,
  "zoneName": "Zone A - Ottapalam North",
  "reportedBy": "Raju K.",
  "contactNumber": "9847000001",
  "affectedPeople": 4,
  "isSos": true,
  "requiresBoat": true,
  "requiresMedical": true
}
```

**Update status example:**
```http
PATCH /api/incidents/1/status
Authorization: Bearer eyJ...

{
  "status": "RESOLVED",
  "updatedBy": "coordinator1",
  "note": "Family evacuated safely. Volunteer RB-04 confirmed."
}
```

---

### Volunteers

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| `POST` | `/volunteers` | ❌ Public | Self-register |
| `GET` | `/volunteers` | 🔑 Coord+ | All volunteers |
| `GET` | `/volunteers/{id}` | ✅ | Single volunteer |
| `GET` | `/volunteers/status/{status}` | 🔑 Coord+ | By status |
| `GET` | `/volunteers/zone/{zone}` | ✅ | By zone |
| `GET` | `/volunteers/nearby-boats` | 🔑 Coord+ | `?lat=&lng=&radius=` |
| `GET` | `/volunteers/stats` | ✅ | Count by status |
| `PATCH` | `/volunteers/{id}/location` | ✅ | GPS check-in |
| `PATCH` | `/volunteers/{id}/status` | ✅ | Update own status |
| `PATCH` | `/volunteers/{id}/zone` | 🔑 Coord+ | Assign zone |

**GPS check-in example:**
```http
PATCH /api/volunteers/3/location
Authorization: Bearer eyJ...

{ "latitude": 10.7870, "longitude": 76.3880 }
```

---

### Resources

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| `POST` | `/resources` | 🔑 Coord+ | Create resource |
| `GET` | `/resources` | ✅ | All resources |
| `GET` | `/resources/{id}` | ✅ | Single resource |
| `GET` | `/resources/category/{cat}` | ✅ | By category |
| `GET` | `/resources/critical` | ✅ | Below critical threshold |
| `GET` | `/resources/low` | ✅ | Below warning threshold |
| `PATCH` | `/resources/{id}/quantity` | 🔑 Field+ | Update available qty |
| `PATCH` | `/resources/{id}/zone` | 🔑 Coord+ | Allocate to zone |

**Resource categories:** `FOOD` · `WATER` · `MEDICAL` · `RESCUE_EQUIPMENT` · `SHELTER` · `VEHICLES` · `POWER` · `COMMUNICATION`

**Update quantity example:**
```http
PATCH /api/resources/4/quantity
Authorization: Bearer eyJ...

{ "availableQuantity": 15, "notes": "Used 5 vials at Shelter 2 (diabetic patients)" }
```

---

### Dashboard

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| `GET` | `/dashboard/summary` | ✅ | Aggregated live stats |
| `GET` | `/dashboard/zones` | ✅ | All flood zones |
| `GET` | `/dashboard/zones/evacuation` | ✅ | Evacuation-ordered zones |

**Summary response:**
```json
{
  "incidents": { "total_active": 5, "critical": 2, "sos_active": 23 },
  "volunteers": { "total": 42, "active": 18, "deployed": 14, "standby": 10 },
  "resources": { "critically_low": 3, "low": 6 },
  "zones": { "total": 4, "critical": 2, "evacuation_required": 2 }
}
```

---

## 📡 WebSocket Topics (STOMP)

**Endpoint:** `ws://localhost:8080/api/ws` (SockJS fallback available)

### Subscribe (dashboard → receive updates)

| Topic | Payload | When |
|---|---|---|
| `/topic/incidents` | `{ type, payload: Incident, timestamp }` | Incident created, updated, assigned, resolved |
| `/topic/volunteers/locations` | `{ volunteerId, lat, lng, name, status }` | Every GPS check-in |
| `/topic/resources` | `{ type: "RESOURCE_UPDATED", payload: Resource }` | Quantity changed |
| `/topic/alerts/resources` | `{ type, resourceId, name, available }` | Critical/low threshold crossed |

### Publish (client → send to server)

| Destination | Payload | Effect |
|---|---|---|
| `/app/sos` | `Incident` object | Creates + broadcasts SOS |
| `/app/volunteer/location` | `{ volunteerId, lat, lng }` | Updates GPS |
| `/app/incident/update` | Any object | Broadcasts to `/topic/incidents` |

**Connect example (JavaScript):**
```javascript
import { Client } from '@stomp/stompjs';
import SockJS from 'sockjs-client';

const client = new Client({
  webSocketFactory: () => new SockJS('http://localhost:8080/api/ws'),
  connectHeaders: { Authorization: `Bearer ${token}` },
  reconnectDelay: 5000,
  onConnect: () => {
    // Receive live incident updates
    client.subscribe('/topic/incidents', (frame) => {
      const update = JSON.parse(frame.body);
      console.log(update.type, update.payload);
    });

    // Send SOS
    client.publish({
      destination: '/app/sos',
      body: JSON.stringify({ title: 'Help needed', severity: 'CRITICAL', isSos: true, latitude: 10.787, longitude: 76.386 })
    });
  }
});

client.activate();
```

---

## 🔐 Environment Variables

| Variable | Default | Notes |
|---|---|---|
| `DB_USERNAME` | `floodguard` | PostgreSQL user |
| `DB_PASSWORD` | `floodguard123` | **Change in production** |
| `KAFKA_SERVERS` | `localhost:9092` | Broker address |
| `JWT_SECRET` | *(see application.yml)* | Min 32 chars, **change in production** |
| `SPRING_DATASOURCE_URL` | `jdbc:postgresql://localhost:5432/floodguard_db` | Full JDBC URL |

**Production `.env` template:**
```env
DB_USERNAME=floodguard_prod
DB_PASSWORD=<strong-random-password>
KAFKA_SERVERS=kafka-broker-1:9092,kafka-broker-2:9092
JWT_SECRET=<64-char-random-hex-string>
SPRING_DATASOURCE_URL=jdbc:postgresql://prod-db-host:5432/floodguard_db
```

---

## 🧪 Tech Stack

### Backend

| Technology | Version | Role |
|---|---|---|
| Java | 17 LTS | Primary language |
| Spring Boot | 3.2.0 | Framework |
| Spring Security | 6.x | JWT auth + RBAC |
| Spring Data JPA | 3.x | ORM + repositories |
| Spring WebSocket | 6.x | STOMP broker |
| Spring Kafka | 3.x | Event streaming |
| PostgreSQL | 15 | Primary database |
| Hibernate Spatial | 6.x | PostGIS spatial queries |
| Apache Kafka | 3.5 | Async event bus |
| jjwt | 0.11.5 | JWT generation/validation |
| Lombok | 1.18.x | Boilerplate reduction |
| MapStruct | 1.5.5 | DTO mapping |

### Frontend

| Technology | Version | Role |
|---|---|---|
| React | 18.2 | UI framework |
| React Router | 6.20 | SPA routing |
| Leaflet + React-Leaflet | 1.9 / 4.2 | Interactive maps |
| @stomp/stompjs | 7.0 | WebSocket client |
| SockJS-client | 1.6 | WS transport fallback |
| Axios | 1.6 | HTTP client |
| Recharts | 2.10 | Charts (dashboard) |
| date-fns | 2.30 | Date formatting |

### Infrastructure

| Technology | Role |
|---|---|
| Docker + Docker Compose | Container orchestration |
| nginx 1.25 | Frontend serving + API proxy |
| Confluent Kafka + Zookeeper | Message broker |
| PostGIS 3.3 | Geospatial extensions for PostgreSQL |

---

## 🗄️ Database Schema

```sql
users            (id, username, password, email, is_active, created_at)
user_roles       (user_id, role)

flood_zones      (id, zone_name, severity, center_lat, center_lng, radius_km,
                  water_level_meters, evacuation_required,
                  estimated_affected_population, active_sos_count, updated_at)

volunteers       (id, full_name, phone, email, status, assigned_zone,
                  current_latitude, current_longitude, last_checkin,
                  is_certified_rescue, is_medical_trained, has_boat,
                  has_vehicle, created_at, updated_at)
volunteer_skills (volunteer_id, skill)

incidents        (id, title, description, severity, status, latitude, longitude,
                  zone_name, reported_by, contact_number, affected_people,
                  is_sos, requires_boat, requires_medical,
                  assigned_volunteer_id, created_at, updated_at, resolved_at)
incident_updates (id, incident_id, message, updated_by, created_at)

resources        (id, name, category, location_name, location_latitude,
                  location_longitude, total_quantity, available_quantity,
                  unit, critical_threshold, warning_threshold,
                  assigned_to_zone, notes, updated_at)
```

**Key spatial query — nearest boat rescuer:**
```sql
SELECT v.* FROM volunteers v
WHERE v.status IN ('ACTIVE', 'STANDBY')
  AND v.has_boat = true
  AND v.current_latitude IS NOT NULL
  AND (
    6371 * acos(
      cos(radians(:lat)) * cos(radians(v.current_latitude)) *
      cos(radians(v.current_longitude) - radians(:lng)) +
      sin(radians(:lat)) * sin(radians(v.current_latitude))
    )
  ) <= :radiusKm
ORDER BY distance ASC
LIMIT 5
```

---

## 🧪 Testing

### Unit tests

```bash
./mvnw test
```

| Test Class | Tests | Coverage |
|---|---|---|
| `IncidentServiceTest` | 4 | SOS Kafka publish, status resolve, volunteer freed, not-found |
| `ResourceServiceTest` | 5 | Critical alert fires, no alert above threshold, utilisation %, isCriticallyLow |

### Manual API testing

Import the following `curl` snippets to test locally:

```bash
# Login
TOKEN=$(curl -s -X POST http://localhost:8080/api/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"admin","password":"admin123"}' | python3 -c "import sys,json; print(json.load(sys.stdin)['token'])")

# Submit SOS (no auth)
curl -X POST http://localhost:8080/api/incidents \
  -H 'Content-Type: application/json' \
  -d '{"title":"Roof collapse risk","severity":"HIGH","latitude":10.787,"longitude":76.386,"isSos":true,"affectedPeople":3}'

# Get active SOS
curl http://localhost:8080/api/incidents/sos -H "Authorization: Bearer $TOKEN"

# Get dashboard summary
curl http://localhost:8080/api/dashboard/summary -H "Authorization: Bearer $TOKEN"

# Volunteer GPS check-in
curl -X PATCH http://localhost:8080/api/volunteers/1/location \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{"latitude":10.787,"longitude":76.386}'

# Find critical resources
curl http://localhost:8080/api/resources/critical -H "Authorization: Bearer $TOKEN"
```

---

## 🚢 Deployment

### Docker Compose (recommended for staging)

```bash
# Build and start everything
docker-compose up --build -d

# View logs
docker-compose logs -f backend

# Stop
docker-compose down

# Stop and remove volumes (wipes database)
docker-compose down -v
```

### Production checklist

- [ ] Change `JWT_SECRET` to a 64+ character random string
- [ ] Change `DB_PASSWORD` to a strong password
- [ ] Set `spring.jpa.hibernate.ddl-auto: validate` (not `update`) in prod
- [ ] Configure Kafka with replication factor ≥ 2
- [ ] Set up PostgreSQL with regular backups
- [ ] Place nginx or a load balancer in front of the backend
- [ ] Enable HTTPS (Let's Encrypt or your CA)
- [ ] Set `CORS` allowed origins to your domain only in `SecurityConfig`
- [ ] Enable Spring Boot Actuator metrics → connect to Prometheus/Grafana

### Push to GitHub

```bash
cd floodguard
git init
git add .
git commit -m "Initial commit: FloodGuard disaster management portal"
git branch -M main
git remote add origin https://github.com/YOUR_USERNAME/floodguard.git
git push -u origin main
```

The static `floodguard.html` can then be deployed to GitHub Pages:
```bash
# Enable GitHub Pages on the repo root → serves floodguard.html at:
# https://YOUR_USERNAME.github.io/floodguard/floodguard.html
```

---

## 🌊 Use Case

Designed for **Kerala's monsoon flood seasons** (June–August), particularly the Palakkad district along the Bharathapuzha River basin. Based on operational patterns from the 2018 Kerala floods.

**Who uses FloodGuard:**

| Role | What they do |
|---|---|
| **Civilians** | Submit SOS from any device — trapped families, medical emergencies, livestock rescue, road closures |
| **Field Volunteers** | Receive assignments, navigate to locations, check in GPS, update incident status |
| **Shelter Coordinators** | Log resource consumption (ORS, food, fuel), request resupply via the platform |
| **District Coordinators** | Watch the live map, override auto-assignments, manage zone severity, broadcast alerts |
| **Admin** | Manage ops users, view analytics, control system-wide settings |

**Why it matters:**
- During the 2018 Kerala floods, informal WhatsApp coordination caused duplicate rescue attempts and missed SOS calls. FloodGuard centralises all SOS intake and dispatch into one auditable system.
- The Haversine auto-dispatch engine ensures the nearest qualified volunteer reaches a trapped family before manual coordination could even begin.

---

## 🤝 Contributing

1. Fork the repository
2. Create a feature branch: `git checkout -b feature/your-feature`
3. Commit with clear messages: `git commit -m "Add: zone severity auto-escalation based on water level"`
4. Push: `git push origin feature/your-feature`
5. Open a pull request

Areas that would benefit from contributions:
- Offline PWA mode for volunteers in low-connectivity areas
- Push notifications (FCM) for volunteer dispatch alerts
- SMS fallback via Twilio for civilians without internet
- Multi-language support (Malayalam, Tamil, Hindi)
- Historical incident analytics dashboard

---

## 📄 License

**MIT** — free to use, modify, and deploy for humanitarian and government purposes.

---

<div align="center">

Built for Kerala's flood responders · Palakkad District Operations

Emergency: **112** · Kerala Disaster Management Authority: **1077**

</div>
