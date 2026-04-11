# Step 11 Phase C — Production Hardening

## Scope

Phase C builds on Phase A (callId, acks, HTTP accept, heartbeat, stale token cleanup, building timeout) and Phase B (durable ringing, retry orchestration, visitor feedback, sleep mode, delivery audit) to add the final production hardening layer.

It covers sub-steps **11.6, 11.7, 11.8, and 11.17**.

After Phase C, the system proactively identifies unhealthy devices before calls are missed, fallback escalation channels reach residents when all push delivery fails, recently-awakened devices can join a still-ringing call, and residents see clear missed-call context when they wake too late.

---

## Prerequisites (Phase A + Phase B — Done)

Phase C depends on all Phase A and Phase B items being in place:

**Phase A:**
- **11.1 + 11.16**: `callId` on every ring, written to `calls` and `audit_logs` tables
- **11.2**: Device acknowledgement events (`call_delivery_acks` table, `/api/home/calls/:callId/ack` endpoint)
- **11.5**: HTTP accept (`/api/home/calls/:callId/accept`)
- **11.13**: Building-level `no_answer_timeout` drives ring expiry
- **11.14**: Stale token cleanup on known-bad push errors
- **11.15**: WebSocket ping/pong heartbeat for intercom connections

**Phase B:**
- **11.12**: Sleep mode respected per-resident before delivery
- **11.3**: Ringing sessions persisted in DB (`call_delivery_attempts`, `calls.expires_at`)
- **11.4**: Retry orchestration for unacknowledged devices (3s / 8s / 15s)
- **11.11**: Intercom visitor feedback (`ring-progress` messages)
- **11.9**: Delivery audit view and portal UI (`call_delivery_summary`)

---

## Implementation Order

Items are ordered so each builds on the previous one. Items 1 and 2 are independent and can start in parallel. Items 3–4 build progressively on the data and infrastructure from earlier items.

| # | Section | Sub-step | Dependencies | Effort |
|---|---------|----------|-------------|--------|
| 1 | C1 | Late-Join and Extended Ring Lifetime (11.6) | Phase A (callId, building timeout), Phase B (durable ringing) | Small |
| 2 | C2 | Missed-Call Recovery UX (11.17) | Phase A (callId), Phase B (calls in DB) | Medium |
| 3 | C3 | Device Health Scoring (11.7) | Phase A (acks), Phase B (delivery attempts, audit view) | Large |
| 4 | C4 | Fallback Escalation Channels (11.8) | C3 (health data), Phase B (retry orchestration, delivery-degraded events) | Large |

---

## C1 — Late-Join and Extended Ring Lifetime (11.6)

### Problem

A device may wake slowly after a VoIP push — the app launches, connects to the signaling server, but by then the pending ring has already expired. The resident sees a brief flash of the incoming call UI that immediately disappears because the server already moved the call to `unanswered`.

With building-level `no_answer_timeout` (Phase A — 11.13) and durable ringing (Phase B — 11.3), the infrastructure exists to support late-joining devices, but the server currently rejects connections from devices that arrive after the ring timer fires.

### What To Do

#### Server — allow late-join on WS connect

When a home client connects via WebSocket and identifies its apartment, the server should check whether there is an active ringing call for that apartment:

**File: `vidrom-signaling-server/src/wsHandler.js`**

In the home client connection handler (where the apartment is identified):

1. Query for an active call:
   ```sql
   SELECT id, status, expires_at FROM calls
   WHERE apartment_id = $1 AND status = 'calling' AND expires_at > NOW()
   ORDER BY created_at DESC LIMIT 1
   ```
2. If a still-ringing call exists, send the client a `ring` message with the `callId`:
   ```json
   { "type": "ring", "callId": "...", "buildingId": "...", "apartmentId": "...", "lateJoin": true }
   ```
3. The home app already handles `ring` messages by transitioning to `incoming` state. The `lateJoin` flag is informational (for logging/analytics).

#### Server — allow late-join when call is already accepted elsewhere

If the call has already been accepted (`status = 'accepted'`), the late-arriving device should receive an immediate `call-taken` instead of a stale ring:

```json
{ "type": "call-taken", "callId": "...", "acceptedBy": "..." }
```

This prevents the device from showing an incoming call UI for a call that was already answered.

#### Home app — handle late-join gracefully

**File: `vidrom-ai-home/useCallManager.js`**

The existing `ring` handler already sets `callState = 'incoming'` and sends an `app-awake` ack. No changes needed for the base case.

For the `call-taken` case on connect, the existing handler already sets `callState = 'idle'` and cleans up. No changes needed.

#### Ring lifetime tuning

The ring lifetime is already driven by the building's `no_answer_timeout` (Phase A — 11.13). No code change needed here, but recommend documenting that building managers can increase this value via the management portal if residents report slow wake times.

A reasonable production default is 30–45 seconds. Buildings with older devices or known slow-wake issues can increase to 60 seconds.

### Files To Touch

| File | Change |
|------|--------|
| `vidrom-signaling-server/src/wsHandler.js` | On home client connect, check for active ringing call and send `ring` or `call-taken` |

### Verification

- [ ] Device that connects during an active ring receives a `ring` message with the call's `callId`
- [ ] Device that connects after call is accepted receives `call-taken` immediately
- [ ] Device that connects after ring expired receives nothing (no stale ring)
- [ ] Late-joining device can accept the call normally (HTTP accept + WS offer flow)
- [ ] `lateJoin: true` is logged for analytics
- [ ] Existing on-time ring behavior is unaffected

---

## C2 — Missed-Call Recovery UX (11.17)

### Problem

If a resident's device wakes too late (after the ring expired and no one answered), the app transitions from `incoming` to `idle` with no context. The resident may have seen a brief notification or CallKit UI that vanished, and has no way to know who rang or when.

With `callId` tracking (Phase A) and call history in the `calls` table (Phase A — 11.16), the data exists to show the resident what happened.

### What To Do

#### Server — missed-call query endpoint

Add an HTTP endpoint for the home app to fetch recent missed calls:

**File: `vidrom-signaling-server/src/httpRoutes.js`**

**GET `/api/home/calls/missed`**

Query parameters: `apartmentId`, `userId`, `since` (ISO timestamp, default last 24 hours)

```sql
SELECT c.id AS call_id, c.status, c.started_at, c.ended_at,
       b.name AS building_name, i.name AS intercom_name
FROM calls c
JOIN buildings b ON b.id = c.building_id
JOIN intercoms i ON i.id = c.intercom_id
WHERE c.apartment_id = $1
  AND c.status = 'unanswered'
  AND c.created_at > $2
ORDER BY c.created_at DESC
LIMIT 20
```

Return:
```json
[
  {
    "callId": "...",
    "status": "unanswered",
    "startedAt": "2026-04-10T14:30:00Z",
    "buildingName": "Sunrise Towers",
    "intercomName": "Main Entrance"
  }
]
```

#### Server — on ring expiry, push missed-call data over WebSocket

When a ring expires with no answer (the `onExpired` callback in `setPendingRing`), send a `missed-call` message to all connected home clients for that apartment:

**File: `vidrom-signaling-server/src/wsHandler.js`**

In the ring expiry callback:
```json
{
  "type": "missed-call",
  "callId": "...",
  "intercomName": "Main Entrance",
  "buildingName": "Sunrise Towers",
  "startedAt": "2026-04-10T14:30:00Z"
}
```

#### Home app — show missed-call context

**File: `vidrom-ai-home/useCallManager.js`**

Handle the `missed-call` message type:

1. If the app is in `incoming` state (showed the ring but timed out), transition to `idle` and display a brief toast or in-app notification: "Missed call from Main Entrance"
2. If the app is in `idle` state (just connected, ring already expired), show the missed call info as a banner or notification

**File: `vidrom-ai-home/HomeScreen.js`**

Show a missed-call indicator on the home screen:

- Display a "Missed Calls" section or badge when there are recent unanswered calls
- Tapping it could show the list from the `/api/home/calls/missed` endpoint
- Dismiss when the user has seen it

#### Home app — on late connect when call is already ended

When the home app connects and the server finds no active ring but there is a recent `unanswered` call for the apartment (within the last few minutes), send the `missed-call` message so the resident knows what happened.

**File: `vidrom-signaling-server/src/wsHandler.js`**

In the home client connect handler (extending C1's late-join check):

```sql
SELECT c.id, c.status, c.started_at, b.name AS building_name, i.name AS intercom_name
FROM calls c
JOIN buildings b ON b.id = c.building_id
JOIN intercoms i ON i.id = c.intercom_id
WHERE c.apartment_id = $1
  AND c.status = 'unanswered'
  AND c.created_at > NOW() - INTERVAL '5 minutes'
ORDER BY c.created_at DESC
LIMIT 1
```

If found, send `missed-call` to the newly connected client.

### Files To Touch

| File | Change |
|------|--------|
| `vidrom-signaling-server/src/httpRoutes.js` | Add `GET /api/home/calls/missed` endpoint |
| `vidrom-signaling-server/src/wsHandler.js` | Send `missed-call` on ring expiry; send on late connect if recent unanswered call |
| `vidrom-ai-home/useCallManager.js` | Handle `missed-call` WS message |
| `vidrom-ai-home/HomeScreen.js` | Display missed-call indicator/banner |

### Verification

- [ ] Ring expires → all connected home clients receive `missed-call` with intercom and building context
- [ ] Device that connects shortly after a missed call receives `missed-call` on connect
- [ ] `/api/home/calls/missed` returns recent unanswered calls for the apartment
- [ ] Home screen shows missed-call indicator
- [ ] Missed calls older than 5 minutes are not pushed on connect (only available via API)
- [ ] Calls that were answered by another resident (`call-taken`) are NOT shown as missed

---

## C3 — Device Health Scoring (11.7)

### Problem

The system currently has no way to know whether an apartment's devices are healthy enough to receive a call *before* the call happens. Stale tokens are cleaned reactively (Phase A — 11.14) and delivery failures are logged (Phase B — 11.9), but there is no proactive health assessment.

A building manager should be able to see which apartments have unhealthy or no devices before a real visitor call fails.

### What To Do

#### New table: `device_health`

Create migration `009-device-health.sql`:

```sql
CREATE TABLE IF NOT EXISTS device_health (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    device_token            TEXT NOT NULL,
    user_id                 UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    apartment_id            UUID NOT NULL REFERENCES apartments(id) ON DELETE CASCADE,
    platform                VARCHAR(10) NOT NULL CHECK (platform IN ('ios', 'android')),
    token_type              VARCHAR(10) NOT NULL CHECK (token_type IN ('fcm', 'voip')),
    
    -- Health signals
    last_successful_push    TIMESTAMPTZ,
    last_push_failure       TIMESTAMPTZ,
    last_push_error         TEXT,
    last_token_refresh      TIMESTAMPTZ,
    last_ack_at             TIMESTAMPTZ,
    last_call_ack_event     VARCHAR(30),
    notification_permission VARCHAR(10) DEFAULT 'unknown'
                            CHECK (notification_permission IN ('granted', 'denied', 'unknown')),
    app_version             VARCHAR(20),
    os_version              VARCHAR(20),
    
    -- Computed score
    health_score            INTEGER DEFAULT 100 CHECK (health_score BETWEEN 0 AND 100),
    health_status           VARCHAR(15) DEFAULT 'unknown'
                            CHECK (health_status IN ('healthy', 'degraded', 'unhealthy', 'unknown')),
    
    last_evaluated_at       TIMESTAMPTZ,
    created_at              TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at              TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    
    UNIQUE (device_token, token_type)
);

CREATE INDEX idx_device_health_apartment ON device_health (apartment_id);
CREATE INDEX idx_device_health_status ON device_health (health_status);
CREATE INDEX idx_device_health_user ON device_health (user_id);
```

#### Apartment health summary view

```sql
CREATE OR REPLACE VIEW apartment_device_health AS
SELECT
    dh.apartment_id,
    a.number AS apartment_number,
    a.building_id,
    COUNT(DISTINCT dh.device_token) AS total_devices,
    COUNT(DISTINCT dh.device_token) FILTER (WHERE dh.health_status = 'healthy') AS healthy_devices,
    COUNT(DISTINCT dh.device_token) FILTER (WHERE dh.health_status = 'degraded') AS degraded_devices,
    COUNT(DISTINCT dh.device_token) FILTER (WHERE dh.health_status = 'unhealthy') AS unhealthy_devices,
    CASE
        WHEN COUNT(DISTINCT dh.device_token) = 0 THEN 'no-devices'
        WHEN COUNT(DISTINCT dh.device_token) FILTER (WHERE dh.health_status = 'healthy') > 0 THEN 'ok'
        WHEN COUNT(DISTINCT dh.device_token) FILTER (WHERE dh.health_status = 'degraded') > 0 THEN 'at-risk'
        ELSE 'critical'
    END AS apartment_health
FROM device_health dh
JOIN apartments a ON a.id = dh.apartment_id
GROUP BY dh.apartment_id, a.number, a.building_id;
```

#### Health score computation

Add a new module `deviceHealthScorer.js`:

**File: `vidrom-signaling-server/src/deviceHealthScorer.js`**

The scorer runs periodically (e.g., every 10 minutes) and evaluates each device:

**Scoring rules (starting from 100, subtract penalties):**

| Signal | Penalty | Rationale |
|--------|---------|-----------|
| No successful push in last 7 days | -30 | Token may be stale |
| Last push failed | -20 | Active delivery problem |
| No ack on last call | -15 | Device may not be displaying notifications |
| Token not refreshed in 30 days | -10 | Token aging risk |
| Notification permission denied | -40 | Cannot deliver at all |
| No ack ever recorded | -10 | Never confirmed working |
| App version older than 2 major versions | -5 | Compatibility risk |

**Health status thresholds:**

| Score | Status |
|-------|--------|
| 80–100 | `healthy` |
| 50–79 | `degraded` |
| 0–49 | `unhealthy` |

The scorer should:

1. Query all active device tokens from `device_tokens`
2. Join with the latest data from `call_delivery_attempts` and `call_delivery_acks`
3. Compute the score for each device
4. Upsert into `device_health`

#### Feed health data from existing events

Update existing handlers to write health signals as they occur:

| Event | Update |
|-------|--------|
| Push sent successfully | `last_successful_push = NOW()` |
| Push failed | `last_push_failure = NOW()`, `last_push_error = error` |
| Delivery ack received | `last_ack_at = NOW()`, `last_call_ack_event = event` |
| Token registered/refreshed | `last_token_refresh = NOW()` |
| App reports permission state | `notification_permission = state` |
| App reports version | `app_version = version`, `os_version = os` |

These are lightweight upserts that happen on existing code paths — no new endpoints needed.

#### Home app — report health signals

**File: `vidrom-ai-home/useCallManager.js` or `vidrom-ai-home/fcmService.js`**

On app launch or WS connect, report device metadata:

```javascript
// On WS connect, send device info
ws.send(JSON.stringify({
  type: 'device-info',
  platform: Platform.OS,
  osVersion: Platform.Version,
  appVersion: Constants.expoConfig?.version,
  notificationPermission: await getNotificationPermissionStatus(),
}));
```

The server stores this in `device_health` for the connected user's devices.

#### Management portal — apartment health dashboard

**File: `vidrom-signaling-server/portals/management.html`**

Add an "Apartment Health" section showing:

- List of apartments with their `apartment_health` status (ok / at-risk / critical / no-devices)
- For `at-risk` and `critical` apartments, show device-level breakdown
- Color coding: green (ok), yellow (at-risk), red (critical), grey (no-devices)

**File: `vidrom-signaling-server/lambda/managementRoutes.js`**

Add endpoint:
- **GET `/api/management/buildings/:buildingId/device-health`** — returns `apartment_device_health` view for the building

#### Admin portal — system health overview

**File: `vidrom-signaling-server/portals/admin.html`**

Add system-wide device health metrics:

- Total devices by health status across all buildings
- Buildings with critical apartments
- Trend of health scores over time

**File: `vidrom-signaling-server/lambda/adminRoutes.js`**

Add endpoint:
- **GET `/api/admin/device-health/summary`** — returns aggregated health stats

### Files To Touch

| File | Change |
|------|--------|
| `vidrom-signaling-server/sql/009-device-health.sql` | New migration — `device_health` table and `apartment_device_health` view |
| `vidrom-signaling-server/src/deviceHealthScorer.js` | New module — periodic health score computation |
| `vidrom-signaling-server/src/server.js` | Start health scorer interval on server boot |
| `vidrom-signaling-server/src/wsHandler.js` | Handle `device-info` message; update health signals on push/ack events |
| `vidrom-signaling-server/src/httpRoutes.js` | Update health signals on token registration and ack |
| `vidrom-signaling-server/lambda/managementRoutes.js` | Add device health endpoint |
| `vidrom-signaling-server/lambda/adminRoutes.js` | Add system health endpoint |
| `vidrom-signaling-server/portals/management.html` | Apartment health dashboard section |
| `vidrom-signaling-server/portals/admin.html` | System health overview section |
| `vidrom-ai-home/useCallManager.js` | Send `device-info` on WS connect |

### Verification

- [ ] `device_health` table is populated after health scorer runs
- [ ] Health score decreases when push fails or no ack is received
- [ ] Health score recovers when successful push/ack occurs
- [ ] `apartment_device_health` view correctly aggregates per apartment
- [ ] Management portal shows apartment health with color coding
- [ ] Apartments with no registered tokens show as `no-devices`
- [ ] Home app reports `device-info` on connect; server stores it
- [ ] Admin portal shows system-wide health metrics
- [ ] Health scorer runs every 10 minutes without blocking the event loop

---

## C4 — Fallback Escalation Channels (11.8)

### Problem

If no device in the apartment acknowledges a ring after all retries (the `delivery-degraded` event from Phase B — 11.4), the visitor is stuck. The intercom shows "No response" and the call ends. There is no alternative way to reach the resident.

This is the last line of defense — it only fires when the entire push delivery pipeline has failed for a specific call.

### What To Do

#### Database — escalation configuration

Create migration `010-escalation-config.sql`:

```sql
CREATE TABLE IF NOT EXISTS escalation_configs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    apartment_id    UUID NOT NULL REFERENCES apartments(id) ON DELETE CASCADE,
    channel         VARCHAR(20) NOT NULL CHECK (channel IN ('sms', 'phone-call', 'concierge')),
    priority        INTEGER NOT NULL DEFAULT 1,
    target          TEXT NOT NULL,  -- phone number, concierge user ID, etc.
    enabled         BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    
    UNIQUE (apartment_id, channel, target)
);

CREATE INDEX idx_escalation_configs_apartment ON escalation_configs (apartment_id);
```

Each apartment can have multiple escalation channels ordered by priority. For example:

| Priority | Channel | Target |
|----------|---------|--------|
| 1 | sms | +972501234567 |
| 2 | phone-call | +972501234567 |
| 3 | concierge | (building concierge user ID) |

#### Escalation orchestrator

Add a new module `escalationOrchestrator.js`:

**File: `vidrom-signaling-server/src/escalationOrchestrator.js`**

Triggered when the retry orchestrator fires the `delivery-degraded` event (no device acked after all retries):

1. Query `escalation_configs` for the apartment, ordered by priority
2. For each enabled channel, attempt escalation:

**SMS escalation:**
- Send an SMS via a third-party API (e.g., Twilio, AWS SNS) with a message like: "Someone is at your door at [Building Name]. Open the Vidrom app to answer."
- Include a deep link: `vidrom://call/{callId}`
- Log the attempt in `audit_logs` with event type `escalation-sms-sent`

**Phone call escalation:**
- Place an automated call via a telephony API with a brief message: "You have a visitor at [Building Name]. Please open the Vidrom app."
- Log as `escalation-phone-sent`

**Concierge escalation:**
- Send a push notification or WS message to the concierge user/device
- The concierge can answer the intercom call on behalf of the apartment or contact the resident by other means
- Log as `escalation-concierge-sent`

3. Track escalation attempts in the `call_delivery_attempts` table with a new delivery state: `escalated`
4. If escalation succeeds (resident opens app and accepts), the normal accept flow handles it
5. If all escalation channels fail or time out, log `escalation-failed` in `audit_logs`

#### Server — integrate with retry orchestrator

**File: `vidrom-signaling-server/src/retryOrchestrator.js`**

In the `checkDeliveryDegraded` function (the t=15s final check):

```javascript
// Existing: log delivery-degraded, send ring-progress noResponse
// New: trigger escalation if configured
const configs = await getEscalationConfigs(apartmentId);
if (configs.length > 0) {
  await triggerEscalation(callId, apartmentId, configs);
}
```

The escalation should only fire if the ring is still active (not yet expired or answered). Check `calls.status = 'calling'` before escalating.

#### Management portal — escalation configuration

**File: `vidrom-signaling-server/portals/management.html`**

Add an "Escalation Settings" section per apartment where building managers can:

- Add/remove escalation channels (SMS, phone, concierge)
- Set phone numbers for SMS and phone escalation
- Enable/disable channels
- Set priority order

**File: `vidrom-signaling-server/lambda/managementRoutes.js`**

Add endpoints:
- **GET `/api/management/apartments/:apartmentId/escalation`** — get escalation configs
- **PUT `/api/management/apartments/:apartmentId/escalation`** — update escalation configs

#### Intercom — show escalation status

When escalation is triggered, send a `ring-progress` update to the intercom:

```json
{ "type": "ring-progress", "callId": "...", "escalating": true, "channel": "sms" }
```

The intercom can show: "Attempting to reach resident via SMS..."

This gives the visitor hope that someone is still being contacted, rather than just "No response."

#### Rate limiting and safety

- Do not escalate the same apartment more than once per call
- Do not escalate more than 3 times per apartment per hour (prevent abuse from repeated intercom rings)
- Do not send SMS or phone calls to the same number more than 5 times per day
- Log all escalation attempts for audit

### Files To Touch

| File | Change |
|------|--------|
| `vidrom-signaling-server/sql/010-escalation-config.sql` | New migration — escalation configuration table |
| `vidrom-signaling-server/src/escalationOrchestrator.js` | New module — escalation channel execution |
| `vidrom-signaling-server/src/retryOrchestrator.js` | Trigger escalation on delivery-degraded |
| `vidrom-signaling-server/src/wsHandler.js` | Send `ring-progress` with escalation status to intercom |
| `vidrom-signaling-server/lambda/managementRoutes.js` | Escalation config CRUD endpoints |
| `vidrom-signaling-server/portals/management.html` | Escalation settings UI per apartment |
| `vidrom-ai-intercom/InCallScreen.js` | Display escalation status to visitor |

### Third-Party Dependencies

Escalation channels require integration with external services:

| Channel | Suggested Service | Notes |
|---------|-------------------|-------|
| SMS | AWS SNS or Twilio | AWS SNS is simpler if already on AWS; Twilio has better delivery reporting |
| Phone call | Twilio | Programmable voice with TTS message |
| Concierge | Internal (WS/push) | No external dependency |

Start with SMS only (highest value, simplest integration), then add phone and concierge as needed.

### Verification

- [ ] `escalation_configs` table stores per-apartment escalation channels
- [ ] Management portal allows configuring escalation channels
- [ ] When delivery-degraded fires and escalation is configured, SMS is sent
- [ ] SMS contains building name and deep link
- [ ] Escalation is logged in `audit_logs`
- [ ] Escalation does not fire if the call was already answered or expired
- [ ] Rate limits prevent excessive escalation (max 3/hour per apartment, 5/day per number)
- [ ] Intercom shows "Attempting to reach resident via SMS..." during escalation
- [ ] Apartments without escalation configs get the existing no-response behavior (no change)

---

## Rollback Safety

All Phase C changes are additive:

- **C1 (Late-Join)**: The late-join check on WS connect is a new query. Removing it returns to the current behavior where late-arriving devices get no ring.
- **C2 (Missed-Call)**: The `missed-call` WS message and API endpoint are new. The home app ignores unknown message types, so removing the server-side code has no effect. The missed-call UI on the home screen can be hidden.
- **C3 (Device Health)**: The `device_health` table and scorer are independent of the call delivery pipeline. Dropping the table and disabling the scorer has no effect on call delivery. Portal UI sections can be hidden.
- **C4 (Escalation)**: Escalation is only triggered when `escalation_configs` has entries for the apartment. An apartment with no configs gets the existing behavior. Disabling the escalation trigger in the retry orchestrator reverts to delivery-degraded logging only.

---

## Success Criteria

Phase C is complete when:

1. Devices that wake slowly can late-join a still-ringing call without the ring disappearing.
2. Residents see clear missed-call context (intercom name, building, timestamp) when they miss a call.
3. Each device has a health score, and apartments with no healthy devices are surfaced in the management portal.
4. Building managers can configure fallback escalation channels (SMS at minimum) per apartment.
5. When all push delivery fails, the system escalates via the configured channels before giving up.
6. All escalation attempts are logged and rate-limited.
7. All changes are additive and can be rolled back by removing the new code/tables.
