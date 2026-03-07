# Business Specifications

## Entities
- Building
- Apartment
- User (with role: admin / manager / resident)
- Intercom
- Call
- Notification
- AuditLog

## System Structure

The system includes buildings.

### Building
Each building has:
- id
- name
- address
- door-code
- manager
- apartments
- intercoms
- door-opening-time (numeric value in seconds - duration the door lock will open when the door code is entered or a resident chooses to open the door)
- no-answer-timeout (numeric value in seconds - duration before an unanswered call auto-ends)
- language
- volume
- brightness
- dark-mode
- sleep-mode

### Apartment
Each apartment has:
- number
- name
- residents (many-to-many via apartment_residents)

### User
All user types (admin, manager, resident) are stored in a single User entity with a role field.

Each user has:
- id
- email
- name
- role (admin / manager / resident)
- push-notification-token (for residents)
- authentication-method (google / apple)
- sleep-mode (boolean - when enabled, resident does not receive incoming call notifications)

### Intercom
Each intercom has:
- id
- building (which building it belongs to)
- name / gate-id
- status (connected / disconnected)
- is-door-open (runtime state)

### Call
Each call has:
- id
- status (calling / accepted / ended / rejected / unanswered)
- building
- apartment
- intercom (which intercom initiated the call)
- start-time
- end-time

### Notification
Each notification has:
- id
- text
- date
- building (scope)

### AuditLog
Each audit log entry has:
- id
- event-type (call-initiated / call-accepted / call-rejected / call-ended / call-unanswered / door-open / access-code-success / access-code-failure / watch-camera-started)
- time
- date
- building-id
- apartment-id (when relevant)
- user-id (when relevant)
- intercom-id (when relevant)
- call-id (for call-related events)
- description (free-text log message)

**Note:** Other properties might be added later.

## Junction Tables

- **apartment_residents** — many-to-many relationship between apartments and users (residents). A resident can belong to multiple apartments across multiple buildings.
- **building_managers** — relationship between buildings and users (managers). Links managers to the buildings they manage.

## Supported Actions

CRUDs (Create, Read, Update, Delete) for all entities:
- Buildings
- Apartments
- Users
- Intercoms
- Notifications

Operational actions:
- Initiate call (intercom → apartment residents)
- Accept call (resident from home app)
- Reject call (resident from home app)
- End call (resident from home app or intercom)
- Open door/gate (resident — from home screen, during call, or during watch)
- Validate access code (visitor at intercom)
- Watch camera (resident — on-demand live view without a call)

## Global Settings

The system has global settings:
- max-call-duration (numeric value in seconds - maximum duration allowed for a call)
- no-answer-timeout (numeric value in seconds - default duration before an unanswered call auto-ends)
- call-polling-interval (numeric value in seconds - how often to poll for call status)
- intercom-polling-interval (numeric value in seconds - how often to poll intercom status)

## Rules

- Every resident can belong to more than one apartment and more than one building
- A call auto-ends after max-call-duration seconds
- A call is marked as unanswered after no-answer-timeout seconds if no resident picks up
- The door automatically re-locks after door-opening-time seconds
- Access code validation opens the door only if the code matches the building's door-code
- A resident with sleep-mode enabled does not receive incoming call notifications
- Only one active call per apartment at a time
- Every event in the system will be audited (see AuditLog entity for structure and event types)

