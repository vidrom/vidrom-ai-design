# C3 — Device Health Scoring (11.7)

## Context

Part of Step 11 Phase C — Production Hardening.

**Dependencies:** Phase A (acks), Phase B (delivery attempts, audit view)
**Effort:** Medium

---

## Problem

The system currently has no way to know whether an apartment's devices are healthy enough to receive a call *before* the call happens. Stale tokens are cleaned reactively (Phase A — 11.14) and delivery failures are logged (Phase B — 11.9), but there is no proactive health assessment.

A building manager should be able to see which apartments have unhealthy or no devices before a real visitor call fails.

## What To Do

### New table: `device_health`

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

### Apartment health summary view

The view applies time-based penalties at read time so scores stay fresh without a periodic scorer:

```sql
CREATE OR REPLACE VIEW apartment_device_health AS
WITH scored AS (
    SELECT
        dh.*,
        GREATEST(0, LEAST(100,
            dh.health_score
            - CASE WHEN dh.last_successful_push < NOW() - INTERVAL '7 days' THEN 30 ELSE 0 END
            - CASE WHEN dh.last_token_refresh < NOW() - INTERVAL '30 days' THEN 10 ELSE 0 END
        )) AS live_score
    FROM device_health dh
),
with_status AS (
    SELECT *,
        CASE
            WHEN live_score >= 80 THEN 'healthy'
            WHEN live_score >= 50 THEN 'degraded'
            ELSE 'unhealthy'
        END AS live_status
    FROM scored
)
SELECT
    s.apartment_id,
    a.number AS apartment_number,
    a.building_id,
    COUNT(DISTINCT s.device_token) AS total_devices,
    COUNT(DISTINCT s.device_token) FILTER (WHERE s.live_status = 'healthy') AS healthy_devices,
    COUNT(DISTINCT s.device_token) FILTER (WHERE s.live_status = 'degraded') AS degraded_devices,
    COUNT(DISTINCT s.device_token) FILTER (WHERE s.live_status = 'unhealthy') AS unhealthy_devices,
    CASE
        WHEN COUNT(DISTINCT s.device_token) = 0 THEN 'no-devices'
        WHEN COUNT(DISTINCT s.device_token) FILTER (WHERE s.live_status = 'healthy') > 0 THEN 'ok'
        WHEN COUNT(DISTINCT s.device_token) FILTER (WHERE s.live_status = 'degraded') > 0 THEN 'at-risk'
        ELSE 'critical'
    END AS apartment_health
FROM with_status s
JOIN apartments a ON a.id = s.apartment_id
GROUP BY s.apartment_id, a.number, a.building_id;
```

### Health score computation — inline on each event

No periodic scorer module. Instead, each event handler computes the score inline when it upserts `device_health`. The logic is a simple helper function in the handler code:

**Instant penalties (applied on write):**

| Signal | Penalty | Rationale |
|--------|---------|-----------|
| Last push failed | -20 | Active delivery problem |
| No ack on last call | -15 | Device may not be displaying notifications |
| Notification permission denied | -40 | Cannot deliver at all |
| No ack ever recorded | -10 | Never confirmed working |

**Time-based penalties (applied at read time in the view):**

| Signal | Penalty | Rationale |
|--------|---------|-----------|
| No successful push in last 7 days | -30 | Token may be stale |
| Token not refreshed in 30 days | -10 | Token aging risk |

**Health status thresholds:**

| Score | Status |
|-------|--------|
| 80–100 | `healthy` |
| 50–79 | `degraded` |
| 0–49 | `unhealthy` |

The helper function takes the current row fields and returns `{ health_score, health_status }`. Each handler calls it and includes the result in its UPSERT. Time-based penalties are layered on top by the `apartment_device_health` view at query time so they stay fresh without a cron.

### Feed health data from existing events

Update existing handlers to write health signals and inline score as they occur:

| Event | Update |
|-------|--------|
| Push sent successfully | `last_successful_push = NOW()`, recompute score |
| Push failed | `last_push_failure = NOW()`, `last_push_error = error`, recompute score |
| Delivery ack received | `last_ack_at = NOW()`, `last_call_ack_event = event`, recompute score |
| Token registered/refreshed | `last_token_refresh = NOW()`, recompute score |
| App reports permission state | `notification_permission = state`, recompute score |
| App reports version | `app_version = version`, `os_version = os` |

These are lightweight upserts that happen on existing code paths — no new endpoints needed.

### Home app — report health signals

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

### Management portal — apartment health dashboard

**File: `vidrom-signaling-server/portals/management.html`**

Add an "Apartment Health" section showing:

- List of apartments with their `apartment_health` status (ok / at-risk / critical / no-devices)
- For `at-risk` and `critical` apartments, show device-level breakdown
- Color coding: green (ok), yellow (at-risk), red (critical), grey (no-devices)

**File: `vidrom-signaling-server/lambda/managementRoutes.js`**

Add endpoint:
- **GET `/api/management/buildings/:buildingId/device-health`** — returns `apartment_device_health` view for the building

### Admin portal — system health overview

**File: `vidrom-signaling-server/portals/admin.html`**

Add a "Device Health" section showing:

- Total devices by health status across all buildings
- Buildings with critical apartments (link through to management portal view)
- Color coding same as management portal

**File: `vidrom-signaling-server/lambda/adminRoutes.js`**

Add endpoint:
- **GET `/api/admin/device-health/summary`** — returns aggregated health stats from `apartment_device_health` view grouped by building

## Files To Touch

| File | Change |
|------|--------|
| `vidrom-signaling-server/sql/009-device-health.sql` | New migration — `device_health` table and `apartment_device_health` view |
| `vidrom-signaling-server/src/wsHandler.js` | Handle `device-info` message; inline health score upsert on push/ack events |
| `vidrom-signaling-server/src/httpRoutes.js` | Inline health score upsert on token registration and ack |
| `vidrom-signaling-server/lambda/managementRoutes.js` | Add device health endpoint |
| `vidrom-signaling-server/lambda/adminRoutes.js` | Add system health endpoint |
| `vidrom-signaling-server/portals/management.html` | Apartment health dashboard section |
| `vidrom-signaling-server/portals/admin.html` | System health overview section |
| `vidrom-ai-home/useCallManager.js` | Send `device-info` on WS connect |

## Verification

- [ ] `device_health` row is created/updated on push success, push failure, ack, and token refresh
- [ ] Health score decreases when push fails or no ack is received
- [ ] Health score recovers when successful push/ack occurs
- [ ] Time-based penalties apply at read time via the view (e.g., no push in 7 days lowers score)
- [ ] `apartment_device_health` view correctly aggregates per apartment
- [ ] Management portal shows apartment health with color coding
- [ ] Apartments with no registered tokens show as `no-devices`
- [ ] Home app reports `device-info` on connect; server stores it
- [ ] Admin portal shows system-wide health metrics per building