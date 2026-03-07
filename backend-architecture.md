# Backend Architecture

## Overview

The Vidrom backend serves four consumers: the Home App, the Intercom App, the Admin Portal, and the Management Portal. The backend is split into two layers based on communication type:

1. **Real-time layer** (EC2) — WebSocket-based signaling for WebRTC calls and live camera
2. **REST/CRUD layer** (EC2 initially, Lambda later) — stateless HTTP APIs for all data operations

Both layers connect to the same PostgreSQL database (RDS).

---

## Architecture Diagram

```
          Admin Portal       Management Portal       Home App          Intercom App
          (admin.html)       (management.html)       (React Native)    (React Native)
              │                    │                    │    │              │    │
              │  REST              │  REST              │REST│  WS         │REST│  WS
              ▼                    ▼                    ▼    │              ▼    │
  ┌─────────────────────────────────────────────┐          │                   │
  │          REST API Layer                     │          │                   │
  │                                             │          │                   │
  │  /api/admin/*      (system admins)          │          │                   │
  │  /api/management/* (building managers)      │          │                   │
  │  /api/app/*        (home app / residents)   │          │                   │
  │  /api/intercom/*   (intercom devices)       │          │                   │
  │                                             │          │                   │
  │  Today: Express on EC2 signaling server     │          │                   │
  │  Future: API Gateway + Lambda               │          │                   │
  └──────────────────┬──────────────────────────┘          │                   │
                     │                                     │                   │
                     │                          ┌──────────┴───────────────────┘
                     │                          │
                     │                          ▼
                     │               ┌──────────────────────────────┐
                     │               │  Signaling Server (EC2)      │
                     │               │                              │
                     │               │  - WebSocket connections     │
                     │               │  - WebRTC offer/answer/ICE   │
                     │               │  - Call state relay           │
                     │               │  - Watch camera sessions     │
                     │               │  - Push notification trigger │
                     │               └──────────┬───────────────────┘
                     │                          │
                     ▼                          ▼
              ┌──────────────────────────────────────┐
              │  RDS PostgreSQL                      │
              │                                      │
              │  buildings, apartments, users,        │
              │  intercoms, calls, audit_logs,        │
              │  notifications, global_settings,      │
              │  apartment_residents, building_managers│
              └──────────────────────────────────────┘
```

---

## Two Backend Layers

### 1. Real-Time Layer (EC2 Signaling Server)

Handles persistent WebSocket connections and latency-critical operations.

| Responsibility | Description |
|---------------|-------------|
| WebSocket connections | Home app and intercom app maintain persistent connections during calls |
| WebRTC signaling | Relay SDP offer/answer and ICE candidates between peers |
| Call state management | Track calling → accepted → ended transitions in real-time |
| Watch camera sessions | On-demand live video stream without a call |
| Push notification trigger | Send FCM/APNs notifications when intercom initiates a call |

**Why EC2**: WebSocket connections are long-lived and stateful. Lambda doesn't support persistent WebSocket connections natively. The signaling server needs sub-100ms latency for WebRTC negotiation.

### 2. REST/CRUD Layer (EC2 now → Lambda later)

Handles stateless HTTP API requests for all data operations.

| Responsibility | Description |
|---------------|-------------|
| CRUD operations | Buildings, apartments, users, intercoms, notifications |
| Authentication | Google/Apple OAuth token verification, device JWT verification |
| Authorization | Role-based access control with building-level scoping |
| Audit logging | Write audit trail for all events |
| Business logic | Door code validation, call history, push token registration |

**Why Lambda (future)**: All REST endpoints are stateless and independently invocable. Traffic is bursty (admins/managers use portals occasionally, apps make periodic requests). Lambda free tier covers the expected volume.

---

## API Prefixes and Auth

All four consumers share the same REST API backend but use different prefixes with different auth and scoping:

| API Prefix | Consumer | Auth Method | Scope |
|-----------|----------|-------------|-------|
| `/api/admin/*` | Admin Portal | Google OAuth, `role = 'admin'` | Full unrestricted access |
| `/api/management/*` | Management Portal | Google OAuth, `role = 'manager'` | Filtered by `building_managers` |
| `/api/app/*` | Home App | Google/Apple OAuth, `role = 'resident'` | Filtered by `apartment_residents` |
| `/api/intercom/*` | Intercom App | Device JWT | Own building only |

---

## REST Endpoints by Consumer

### Admin Portal (`/api/admin/*`)

Full CRUD on all entities. See [admin-portal.md](../admin-portal.md) for complete list.

### Management Portal (`/api/management/*`)

Scoped CRUD on assigned buildings. See [management-portal.md](../management-portal.md) for complete list.

### Home App (`/api/app/*`)

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/app/buildings` | List buildings the resident belongs to |
| GET | `/api/app/buildings/:id` | Get building details (name, address) |
| GET | `/api/app/apartments` | List resident's apartments |
| GET | `/api/app/apartments/:id/residents` | List co-residents in an apartment |
| GET | `/api/app/calls` | Call history for the resident |
| GET | `/api/app/notifications` | Notifications for resident's buildings |
| PUT | `/api/app/profile` | Update push token, sleep mode |
| POST | `/api/app/door/open` | Open door (outside of a call, by building/intercom ID) |

### Intercom App (`/api/intercom/*`)

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/intercom/authenticate` | Validate provisioning code → receive device JWT |
| GET | `/api/intercom/building` | Get building config (door code, timeouts, language, volume, brightness, etc.) |
| GET | `/api/intercom/apartments` | List apartments + resident display names for call directory |
| POST | `/api/intercom/door/validate-code` | Validate door access code entered by visitor |
| POST | `/api/intercom/door/open` | Open door (after code validated) |
| POST | `/api/intercom/heartbeat` | Report alive status |

---

## Signaling Server WebSocket Messages

These stay on the EC2 signaling server and are NOT part of the REST API:

| Message | Direction | Description |
|---------|-----------|-------------|
| `ring` | Intercom → Server → Home (via push) | Initiate a call |
| `offer` | Intercom → Server → Home | WebRTC SDP offer |
| `answer` | Home → Server → Intercom | WebRTC SDP answer |
| `ice-candidate` | Both → Server → Both | ICE candidate exchange |
| `accept` | Home → Server → Intercom | Call accepted |
| `decline` | Home → Server → Intercom | Call declined |
| `end-call` | Either → Server → Other | Call ended |
| `watch-start` | Home → Server → Intercom | Start live camera view |
| `watch-stop` | Home → Server → Intercom | Stop live camera view |

---

## Interaction Between Layers

The signaling server (real-time layer) sometimes needs to interact with the database:

| Event | Action |
|-------|--------|
| Call initiated | Write `calls` row with status `calling`, write `audit_logs` |
| Call accepted/rejected/ended | Update `calls` row, write `audit_logs` |
| Call unanswered (timeout) | Update `calls` row, write `audit_logs` |
| Door opened during call | Write `audit_logs` |
| Watch camera started | Write `audit_logs` |

**Implementation options:**
1. **Direct DB access** — Signaling server connects to RDS directly (simplest)
2. **Internal API call** — Signaling server calls the REST API (Lambda) internally (cleaner separation but adds latency)

**Recommendation**: Direct DB access from the signaling server. It's simpler and avoids a network hop for latency-critical call events. The signaling server and RDS are in the same VPC.

---

## Current vs Future Deployment

### Current (Development Phase)

Everything runs on EC2 in the signaling server:

```
EC2 Instance
├── WebSocket signaling
├── REST API (/api/admin/*, /api/management/*, /api/app/*, /api/intercom/*)
└── Static file serving (admin.html, management.html)
```

### Future (Production)

```
S3 + CloudFront         → admin.html, management.html
API Gateway + Lambda    → /api/admin/*, /api/management/*, /api/app/*, /api/intercom/*
EC2 Signaling Server    → WebSocket only (+ direct DB writes for audit/call logs)
RDS PostgreSQL          → shared database
```

See [migrate-portals-to-lambda.md](steps/todo-future/migrate-portals-to-lambda.md) for migration details.
