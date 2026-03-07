# Admin Portal (System Administrators)

## Overview

The Admin Portal is a web-based interface for **system administrators** who have full, unrestricted access to the entire Vidrom platform. It is served from the signaling server at `/admin`.

## Access Control

- **Role**: `admin` (from users table)
- **Authentication**: Google OAuth (Google Identity Services)
- **Authorization**: Server-side verification — user must exist in the `users` table with `role = 'admin'`
- **URL**: `https://<server>/admin`

## Capabilities

System administrators have full access to all entities across all buildings.

### Buildings Management

| Action | Description |
|--------|-------------|
| List all buildings | View every building in the system |
| Create building | Add a new building with name, address, door code, settings |
| Edit building | Update any building's settings (name, address, door code, door opening time, no-answer timeout, language, volume, brightness, dark mode, sleep mode) |
| Delete building | Remove a building and all associated data (apartments, intercoms, etc.) |

### Apartments Management

| Action | Description |
|--------|-------------|
| List apartments | View all apartments across all buildings |
| Create apartment | Add an apartment to any building |
| Edit apartment | Update apartment number/name |
| Delete apartment | Remove an apartment and unlink its residents |

### Users Management

| Action | Description |
|--------|-------------|
| List all users | View every user (admins, managers, residents) |
| Create user | Add a new user with any role |
| Edit user | Update user details and role |
| Delete user | Remove a user from the system |
| Assign manager to building | Link a manager user to a building via `building_managers` |
| Remove manager from building | Unlink a manager from a building |
| Assign resident to apartment | Link a resident user to an apartment via `apartment_residents` |
| Remove resident from apartment | Unlink a resident from an apartment |

### Intercoms Management

| Action | Description |
|--------|-------------|
| List all intercoms | View every intercom across all buildings |
| Create intercom | Register a new intercom device for any building |
| Edit intercom | Update intercom name/gate ID |
| Delete intercom | Remove an intercom from the system |
| Provision intercom | Generate a provisioning code for device setup |
| Revoke intercom | Revoke an intercom's access |
| View connection status | See real-time connected/disconnected status |

### Notifications Management

| Action | Description |
|--------|-------------|
| List all notifications | View notifications across all buildings |
| Create notification | Send a notification scoped to any building |
| Delete notification | Remove a notification |

### Audit Logs

| Action | Description |
|--------|-------------|
| View all audit logs | Browse the full audit trail across all buildings |
| Filter audit logs | Filter by event type, building, apartment, user, intercom, date range |

### Global Settings

| Action | Description |
|--------|-------------|
| View global settings | See all system-wide settings |
| Edit global settings | Modify max call duration, no-answer timeout, polling intervals, etc. |

### Dashboard (Future)

- System-wide statistics (total buildings, apartments, users, intercoms)
- Active calls across the system
- Intercom connection status overview
- Recent audit log entries

## API Routes

All Admin Portal API calls use the `/api/admin/*` prefix.

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/admin/buildings` | List all buildings |
| POST | `/api/admin/buildings` | Create building |
| PUT | `/api/admin/buildings/:id` | Update building |
| DELETE | `/api/admin/buildings/:id` | Delete building |
| GET | `/api/admin/buildings/:id/apartments` | List apartments in building |
| POST | `/api/admin/buildings/:id/apartments` | Create apartment |
| PUT | `/api/admin/apartments/:id` | Update apartment |
| DELETE | `/api/admin/apartments/:id` | Delete apartment |
| GET | `/api/admin/users` | List all users |
| POST | `/api/admin/users` | Create user |
| PUT | `/api/admin/users/:id` | Update user |
| DELETE | `/api/admin/users/:id` | Delete user |
| POST | `/api/admin/buildings/:id/managers` | Assign manager to building |
| DELETE | `/api/admin/buildings/:bid/managers/:uid` | Remove manager from building |
| POST | `/api/admin/apartments/:id/residents` | Assign resident to apartment |
| DELETE | `/api/admin/apartments/:aid/residents/:uid` | Remove resident from apartment |
| GET | `/api/admin/devices` | List all intercoms |
| POST | `/api/admin/devices` | Create intercom (returns provisioning code) |
| POST | `/api/admin/devices/:id/revoke` | Revoke intercom |
| GET | `/api/admin/notifications` | List all notifications |
| POST | `/api/admin/notifications` | Create notification |
| DELETE | `/api/admin/notifications/:id` | Delete notification |
| GET | `/api/admin/audit-logs` | List audit logs (with filters) |
| GET | `/api/admin/settings` | Get global settings |
| PUT | `/api/admin/settings/:key` | Update a global setting |

## Technical Details

- **Frontend**: Single HTML file (`admin.html`) with vanilla JS — no build step
- **Served from**: Signaling server at route `/admin` (current); S3 + CloudFront (future)
- **Backend**: REST API on signaling server (current); API Gateway + Lambda (future)
- **Auth flow**: Google Identity Services → ID token → sent as `Authorization: Bearer <token>` on every API call
- **Server-side auth**: `verifyAdminToken()` verifies Google ID token and checks `role = 'admin'` in users table
- **No client-side email whitelist** — authorization is fully server-side based on the users table role field

See [backend-architecture.md](backend-architecture.md) for the full architecture overview.
