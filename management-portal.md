# Management Portal (Building Managers)

## Overview

The Management Portal is a web-based interface for **building managers** who can manage only the buildings assigned to them. It is served from the signaling server at `/management`.

## Access Control

- **Role**: `manager` (from users table)
- **Authentication**: Google OAuth (Google Identity Services)
- **Authorization**: Server-side verification — user must exist in the `users` table with `role = 'manager'`, and every query is scoped to buildings linked via the `building_managers` junction table
- **URL**: `https://<server>/management`

## Scoping Rule

**All data is filtered by the manager's assigned buildings.** A manager can only see and modify entities that belong to buildings where they are listed in the `building_managers` table. The server enforces this on every API call — the client never receives data from unassigned buildings.

## Capabilities

### Buildings (Read-Only + Settings)

| Action | Description |
|--------|-------------|
| List assigned buildings | View only buildings assigned to this manager |
| Edit building settings | Update settings for assigned buildings (door code, door opening time, no-answer timeout, language, volume, brightness, dark mode, sleep mode) |

**Cannot**: Create buildings, delete buildings, or change building name/address (system admin only).

### Apartments Management

| Action | Description |
|--------|-------------|
| List apartments | View apartments in assigned buildings only |
| Create apartment | Add an apartment to an assigned building |
| Edit apartment | Update apartment number/name in assigned buildings |
| Delete apartment | Remove an apartment from an assigned building |

### Residents Management

| Action | Description |
|--------|-------------|
| List residents | View residents in apartments of assigned buildings |
| Assign resident to apartment | Link a resident to an apartment in an assigned building |
| Remove resident from apartment | Unlink a resident from an apartment in an assigned building |

**Cannot**: Create/delete user accounts or change user roles (system admin only).

### Intercoms Management

| Action | Description |
|--------|-------------|
| List intercoms | View intercoms in assigned buildings only |
| Create intercom | Register a new intercom for an assigned building |
| Provision intercom | Generate a provisioning code for an intercom in an assigned building |
| Revoke intercom | Revoke an intercom in an assigned building |
| View connection status | See real-time status of intercoms in assigned buildings |

**Cannot**: Delete intercoms (system admin only).

### Notifications Management

| Action | Description |
|--------|-------------|
| List notifications | View notifications for assigned buildings only |
| Create notification | Send a notification to an assigned building |
| Delete notification | Remove a notification from an assigned building |

### Audit Logs (Read-Only)

| Action | Description |
|--------|-------------|
| View audit logs | Browse audit logs for assigned buildings only |
| Filter audit logs | Filter by event type, apartment, user, intercom, date range — within assigned buildings |

### Not Available to Managers

| Feature | Reason |
|---------|--------|
| Create/delete buildings | System-level operation |
| Manage user accounts | System-level operation |
| Assign/remove managers | System-level operation |
| Global settings | System-level operation |
| View data from other buildings | Scoping enforcement |
| Delete intercoms | System-level operation |

## API Routes

All Management Portal API calls use the `/api/management/*` prefix. Every endpoint automatically scopes results to the authenticated manager's assigned buildings.

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/management/buildings` | List assigned buildings |
| PUT | `/api/management/buildings/:id` | Update building settings (scoped) |
| GET | `/api/management/buildings/:id/apartments` | List apartments (scoped) |
| POST | `/api/management/buildings/:id/apartments` | Create apartment (scoped) |
| PUT | `/api/management/apartments/:id` | Update apartment (scoped) |
| DELETE | `/api/management/apartments/:id` | Delete apartment (scoped) |
| GET | `/api/management/apartments/:id/residents` | List residents in apartment (scoped) |
| POST | `/api/management/apartments/:id/residents` | Assign resident to apartment (scoped) |
| DELETE | `/api/management/apartments/:aid/residents/:uid` | Remove resident from apartment (scoped) |
| GET | `/api/management/devices` | List intercoms (scoped) |
| POST | `/api/management/devices` | Create intercom (scoped) |
| POST | `/api/management/devices/:id/revoke` | Revoke intercom (scoped) |
| GET | `/api/management/notifications` | List notifications (scoped) |
| POST | `/api/management/notifications` | Create notification (scoped) |
| DELETE | `/api/management/notifications/:id` | Delete notification (scoped) |
| GET | `/api/management/audit-logs` | List audit logs (scoped) |

## Scoping Implementation

Every `/api/management/*` handler must:

1. Extract the authenticated user's ID from the verified Google token + users table
2. Query `building_managers` to get the list of `building_id` values for this user
3. Filter all database queries to only include rows matching those building IDs
4. Return `403 Forbidden` if the manager tries to access a resource outside their assigned buildings

Example server-side pseudocode:
```
const managerBuildings = await db.query(
  'SELECT building_id FROM building_managers WHERE user_id = $1',
  [userId]
);
const buildingIds = managerBuildings.map(r => r.building_id);

// All subsequent queries include: WHERE building_id = ANY($1::uuid[])
```

## UI Differences from Admin Portal

| Element | Admin Portal | Management Portal |
|---------|-------------|-------------------|
| Header color | Dark (#2d3436) | Navy blue (#0c2461) — visual distinction |
| Title | "Vidrom Admin Portal" | "Vidrom Management Portal" |
| Building selector | N/A (sees all) | Dropdown to filter by assigned building |
| Buildings section | Full CRUD | Read + edit settings only |
| Users section | Full CRUD + role management | Not shown |
| Global settings | Shown | Not shown |
| Manager assignment | Shown | Not shown |

## Technical Details

- **Frontend**: Single HTML file (`management.html`) with vanilla JS — no build step
- **Served from**: Signaling server at route `/management` (current); S3 + CloudFront (future)
- **Backend**: REST API on signaling server (current); API Gateway + Lambda (future)
- **Auth flow**: Google Identity Services → ID token → sent as `Authorization: Bearer <token>` on every API call
- **Server-side auth**: `verifyManagementToken()` verifies Google ID token and checks `role = 'manager'` in users table, then loads assigned buildings from `building_managers`
- **No client-side email whitelist** — authorization is fully server-side based on the users table role field and building_managers assignments

See [backend-architecture.md](backend-architecture.md) for the full architecture overview.
