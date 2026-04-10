# Step 11 Phase B — Durable Delivery Workflow

## Scope

Phase B builds on the Phase A foundation (callId, device acks, HTTP accept, building timeout, stale token cleanup, heartbeat) to make call delivery durable, retry-capable, and observable.

It covers sub-steps **11.3, 11.4, 11.9, 11.11, and 11.12**.

After Phase B, ringing sessions survive server restarts, unacknowledged devices are retried automatically, visitors see real-time delivery feedback on the intercom, sleep mode is respected per-resident, and the team has delivery audit data to diagnose regressions.

---

## Prerequisites (Phase A — Done)

Phase B depends on every Phase A item being in place:

- **11.1 + 11.16**: `callId` on every ring, written to `calls` and `audit_logs` tables
- **11.2**: Device acknowledgement events (`call_delivery_acks` table, `/api/home/calls/:callId/ack` endpoint)
- **11.5**: HTTP accept (`/api/home/calls/:callId/accept`)
- **11.13**: Building-level `no_answer_timeout` drives ring expiry
- **11.14**: Stale token cleanup on known-bad push errors
- **11.15**: WebSocket ping/pong heartbeat for intercom connections

---

## Implementation Order

Items are ordered so each builds on the previous one. Item 1 (sleep mode) is a small standalone change. Item 2 (persist ringing) is the foundation for retries. Items 3–5 depend on durable state and acks.

| # | File | Sub-step | Dependencies | Effort |
|---|------|----------|-------------|--------|
| 1 | [step11-B1-sleep-mode.md](step11-B1-sleep-mode.md) | Respect Per-User Sleep Mode (11.12) | Phase A (callId) | Small |
| 2 | [step11-B2-persist-ringing.md](step11-B2-persist-ringing.md) | Persist Ringing Sessions in DB (11.3) | Phase A (callId, acks) | Medium |
| 3 | [step11-B3-retry-orchestration.md](step11-B3-retry-orchestration.md) | Retry Orchestration for Unacknowledged Devices (11.4) | B2 (durable state) + Phase A (acks) | Large |
| 4 | [step11-B4-visitor-feedback.md](step11-B4-visitor-feedback.md) | Intercom Visitor Feedback During Ringing (11.11) | B3 (retry acks flowing) | Medium |
| 5 | [step11-B5-delivery-audit.md](step11-B5-delivery-audit.md) | Delivery Audit and Monitoring (11.9) | B2 + Phase A (acks, callId) | Medium |

---

## B1 — Respect Per-User Sleep Mode in Delivery Workflow (11.12)

### Problem

The `users` table already has a `sleep_mode` boolean, but the ring handler in `wsHandler.js` does not check it. Push notifications are sent to all `device_tokens` for the apartment regardless. A sleeping resident gets woken up, and a visitor rings into the void if all residents have sleep mode on.

### What To Do

#### Server — ring handler (`wsHandler.js`)

When the intercom sends a `ring`, **before** sending push notifications:

1. Query users with sleep mode status for the target apartment:
   ```sql
   SELECT u.id, u.sleep_mode
   FROM users u
   JOIN apartment_residents ar ON ar.user_id = u.id
   WHERE ar.apartment_id = $1
   ```
2. Collect the set of sleeping user IDs.
3. If **all** residents have `sleep_mode = true`, do not ring at all:
   - Do not create a `calls` row with status `calling`
   - Send the intercom an immediate response: `{ type: 'apartment-unavailable', reason: 'all-residents-sleeping', apartmentId }`
   - Break out of the ring handler
4. If **some** residents are sleeping, filter the push token query to exclude sleeping users:
   ```sql
   SELECT dt.token, dt.token_type, dt.platform
   FROM device_tokens dt
   WHERE dt.apartment_id = $1
     AND dt.user_id NOT IN (SELECT u.id FROM users u WHERE u.sleep_mode = true AND u.id = dt.user_id)
   ```
   Or simpler: join with users and filter `WHERE u.sleep_mode = false`.
5. WebSocket ring messages to connected home clients should also be skipped for sleeping users. This requires knowing which connected home client belongs to which user. If user identity is not tracked on the WS connection, send the ring to all connected WS clients but let the home app handle sleep mode locally (lower priority — push filtering is the main win).

#### Intercom — `InCallScreen.js`

Handle the new `apartment-unavailable` message:
- If `reason === 'all-residents-sleeping'`, display "Residents are currently unavailable" instead of "Ringing..."
- After a short delay (3–5 seconds), return to the apartment list or home screen

#### Audit

- Log skipped deliveries in `audit_logs` with event type `ring-skipped-sleep-mode` so the team knows how often this fires.

### Files To Touch

| File | Change |
|------|--------|
| `vidrom-signaling-server/src/wsHandler.js` | Query sleep mode before push fanout; send `apartment-unavailable` if all sleeping |
| `vidrom-ai-intercom/InCallScreen.js` | Handle `apartment-unavailable` WS message |

### Verification

- [ ] If all residents have `sleep_mode = true`, intercom receives `apartment-unavailable` immediately
- [ ] If some residents are sleeping, only awake residents' devices receive push
- [ ] `audit_logs` records `ring-skipped-sleep-mode` when applicable
- [ ] Normal ringing is unaffected when no residents are in sleep mode

---

## B2 — Persist Ringing Sessions in DB (11.3)

### Problem

The current pending ring state lives entirely in memory (`pendingRings` Map in `connectionState.js`). If the signaling server restarts mid-ring, the ring is lost — the intercom thinks it's ringing, but the server has no record. Retries (B3) need durable state to know which devices to retry.

### What To Do

#### New table: `call_delivery_attempts`

Create migration `007-call-delivery-attempts.sql`:

```sql
CREATE TABLE IF NOT EXISTS call_delivery_attempts (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    call_id           UUID NOT NULL REFERENCES calls(id) ON DELETE CASCADE,
    user_id           UUID REFERENCES users(id) ON DELETE SET NULL,
    device_token      TEXT NOT NULL,
    token_type        VARCHAR(10) NOT NULL CHECK (token_type IN ('fcm', 'voip')),
    platform          VARCHAR(10) NOT NULL CHECK (platform IN ('ios', 'android')),
    attempt_number    INTEGER NOT NULL DEFAULT 1,
    delivery_state    VARCHAR(30) NOT NULL DEFAULT 'queued'
                      CHECK (delivery_state IN (
                          'queued', 'push-sent', 'push-failed',
                          'push-received', 'app-awake', 'incoming-ui-shown',
                          'accepted', 'declined', 'timed-out',
                          'skipped-sleep-mode'
                      )),
    last_error        TEXT,
    last_attempt_at   TIMESTAMPTZ,
    acked_at          TIMESTAMPTZ,
    created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_call_delivery_attempts_call ON call_delivery_attempts (call_id);
CREATE INDEX idx_call_delivery_attempts_state ON call_delivery_attempts (call_id, delivery_state);
```

#### Extend `calls` table

Add an `expires_at` column so the server knows when a ring should time out, even after restart:

```sql
ALTER TABLE calls ADD COLUMN IF NOT EXISTS expires_at TIMESTAMPTZ;
```

#### Server — ring handler (`wsHandler.js`)

When a ring starts:

1. Insert into `calls` with `expires_at = NOW() + building_no_answer_timeout`.
2. For each device token being sent a push, insert a row into `call_delivery_attempts` with `delivery_state = 'queued'`.
3. After each push send succeeds, update the row to `push-sent` with `last_attempt_at = NOW()`.
4. If push fails (non-stale error), update to `push-failed` with `last_error`.
5. If push fails with a known-bad token, update to `push-failed`, delete the token (existing behavior), and note the error.

#### Server — ack handler (`httpRoutes.js`)

When a delivery ack arrives at `/api/home/calls/:callId/ack`:

1. Continue writing to `call_delivery_acks` (existing behavior — this is the immutable event log).
2. Also update the corresponding `call_delivery_attempts` row's `delivery_state` and `acked_at`:
   ```sql
   UPDATE call_delivery_attempts
   SET delivery_state = $1, acked_at = NOW()
   WHERE call_id = $2 AND device_token = $3 AND attempt_number = (
       SELECT MAX(attempt_number) FROM call_delivery_attempts
       WHERE call_id = $2 AND device_token = $3
   )
   ```

#### Server — startup recovery

On server start, query for calls that are still in `calling` status and not yet expired:

```sql
SELECT * FROM calls WHERE status = 'calling' AND expires_at > NOW()
```

For each such call, reconstruct the in-memory `pendingRing` and `activeCall` state from the DB. This makes ringing survive a restart.

If a call's `expires_at` has passed during the downtime, update it to `unanswered`.

### Files To Touch

| File | Change |
|------|--------|
| `vidrom-signaling-server/sql/007-call-delivery-attempts.sql` | New migration |
| `vidrom-signaling-server/src/wsHandler.js` | Write delivery attempts on ring; update on push result |
| `vidrom-signaling-server/src/httpRoutes.js` | Update delivery attempts on ack |
| `vidrom-signaling-server/src/connectionState.js` | Add startup recovery from DB |
| `vidrom-signaling-server/src/server.js` | Call recovery on startup after DB connection |

### Verification

- [ ] Each push send creates a `call_delivery_attempts` row
- [ ] Push success → `push-sent`, push failure → `push-failed` with error
- [ ] Ack endpoint updates the latest attempt row's `delivery_state`
- [ ] Server restart during active ring → ring resumes from DB state
- [ ] Expired calls during downtime are marked `unanswered` on startup
- [ ] `calls.expires_at` is set correctly using building timeout

---

## B3 — Retry Orchestration for Unacknowledged Devices (11.4)

### Problem

Phase A sends push notifications once and hopes for the best. If a push is silently dropped by the OS, or a device is slow to wake, there is no retry. The server now has ack data (Phase A) and durable state (B2), so it can identify unacknowledged devices and retry.

### What To Do

#### Retry timeline

After the initial ring push at `t=0`:

| Time | Action |
|------|--------|
| `t=0s` | Send WS ring + push to all eligible devices. Insert `call_delivery_attempts` rows. |
| `t=3s` | **Retry 1**: For devices with no ack (`delivery_state` still `push-sent`), resend push. Update `attempt_number` to 2. |
| `t=8s` | **Retry 2**: For devices still unacked, resend push. Update `attempt_number` to 3. Log a warning per apartment. |
| `t=15s` | **Final check**: If zero devices have acked, log `delivery-degraded` for this call. (Escalation to fallback channels is Phase C.) |

The retry window must stay within the building's `no_answer_timeout`. If timeout is 30s, retries at 3s, 8s, and 15s are safe. If timeout is shorter (e.g., 20s), skip the 15s retry.

#### Server — retry orchestrator

Add a retry scheduler in a new module `retryOrchestrator.js`:

1. When a ring starts, schedule retry checks at the defined intervals using `setTimeout`.
2. At each retry tick:
   - Query `call_delivery_attempts` for this `call_id` where `delivery_state = 'push-sent'` (never acked).
   - For each unacked device, resend the same push (FCM or VoIP).
   - Insert a new `call_delivery_attempts` row with incremented `attempt_number`.
3. Cancel all pending retries when:
   - The call is accepted (any device)
   - The call is declined (all residents)
   - The ring expires (timeout)
   - The intercom hangs up

#### Idempotency

The home app already handles duplicate ring pushes gracefully (it checks if it's already showing the incoming call UI). Retried pushes carry the same `callId`, so the app can deduplicate.

#### Delivery-degraded logging

If no device acks by the final retry check, insert an `audit_logs` entry:
```
event_type: 'delivery-degraded'
description: 'No device acknowledged ring for call <callId> after 3 attempts'
```

This feeds into the monitoring dashboard (B5).

### Files To Touch

| File | Change |
|------|--------|
| `vidrom-signaling-server/src/retryOrchestrator.js` | New module — retry scheduling and execution |
| `vidrom-signaling-server/src/wsHandler.js` | Call retry orchestrator on ring start; cancel on accept/decline/hangup/timeout |
| `vidrom-signaling-server/src/connectionState.js` | Expose retry cancellation on call clear |

### Verification

- [ ] Devices that ack within 3s are not retried
- [ ] Unacked devices get a retry push at ~3s and ~8s
- [ ] Retry inserts new `call_delivery_attempts` rows with incremented `attempt_number`
- [ ] Retries stop immediately when call is accepted, declined, or hung up
- [ ] `delivery-degraded` audit log fires when zero devices ack after all retries
- [ ] Duplicate ring pushes are handled gracefully by the home app
- [ ] Retry intervals respect the building's `no_answer_timeout`

---

## B4 — Intercom Visitor Feedback During Ringing (11.11)

### Problem

The intercom `InCallScreen` shows only "Ringing..." while the visitor waits. The visitor has no idea whether anyone was actually reached. With device acks flowing (Phase A) and retries (B3), the server now has real-time delivery progress it can push to the intercom.

### What To Do

#### Server — ring progress events

Push `ring-progress` messages to the intercom's WebSocket as delivery data comes in:

1. **After initial push fanout** (in the ring handler):
   ```json
   { "type": "ring-progress", "callId": "...", "devicesNotified": 3 }
   ```
2. **When a device ack arrives** (in the ack HTTP handler or via a callback):
   ```json
   { "type": "ring-progress", "callId": "...", "devicesConfirmed": 1 }
   ```
   Send this on the first ack only (or update the count on each subsequent ack).
3. **If no device acks after the retry window** (from the retry orchestrator):
   ```json
   { "type": "ring-progress", "callId": "...", "noResponse": true }
   ```

To send these, the ack handler needs to look up the call's `intercom_id` and find the intercom's WS connection.

#### Intercom — `InCallScreen.js`

Update the ringing UI based on `ring-progress` messages:

| State | Display |
|-------|---------|
| Default (ring sent) | "Ringing apartment 5..." |
| `devicesNotified: N` | "Ringing apartment 5... (N devices notified)" |
| `devicesConfirmed: N` | "Ringing apartment 5... (N devices confirmed)" |
| `noResponse: true` | "No response from apartment 5 — try again or use door code" |

The `noResponse` state should remain until the ring timeout expires and the normal no-answer flow triggers.

#### Implementation notes

- The `ring-progress` messages are informational — they don't affect call state.
- The intercom should handle missing `ring-progress` gracefully (just keep showing "Ringing...").
- `devicesNotified` counts actual push sends (excluding skipped sleep-mode devices).
- `devicesConfirmed` counts distinct devices that sent at least one ack event.

### Files To Touch

| File | Change |
|------|--------|
| `vidrom-signaling-server/src/wsHandler.js` | Send `ring-progress` after push fanout |
| `vidrom-signaling-server/src/httpRoutes.js` | On ack, send `ring-progress` with updated confirmed count to intercom |
| `vidrom-signaling-server/src/retryOrchestrator.js` | Send `ring-progress` with `noResponse` after final retry |
| `vidrom-ai-intercom/InCallScreen.js` | Display delivery progress to visitor |

### Verification

- [ ] Intercom receives `devicesNotified` shortly after ring starts
- [ ] Intercom receives `devicesConfirmed` when first device acks
- [ ] Intercom receives `noResponse` if no device acks after retry window
- [ ] InCallScreen displays updated text for each progress state
- [ ] Missing `ring-progress` messages don't break the default "Ringing..." UI

---

## B5 — Delivery Audit and Monitoring (11.9)

### Problem

The team has no way to see delivery quality across calls. Individual call records exist in `calls` and `audit_logs`, and per-device ack data exists in `call_delivery_acks` and `call_delivery_attempts`, but there's no aggregated view. Delivery regressions (e.g., a broken APNs cert, a new Android OS version suppressing notifications) go unnoticed until residents complain.

### What To Do

#### Delivery summary view

Create a SQL view (or materialized view) that aggregates delivery data per call:

```sql
CREATE OR REPLACE VIEW call_delivery_summary AS
SELECT
    c.id AS call_id,
    c.building_id,
    c.apartment_id,
    c.intercom_id,
    c.status AS call_status,
    c.created_at AS call_started_at,
    c.ended_at AS call_ended_at,
    COUNT(DISTINCT cda.device_token) AS devices_targeted,
    COUNT(DISTINCT cda.device_token) FILTER (WHERE cda.delivery_state IN ('push-received', 'app-awake', 'incoming-ui-shown', 'accepted')) AS devices_acked,
    COUNT(DISTINCT cda.device_token) FILTER (WHERE cda.delivery_state = 'push-failed') AS devices_failed,
    COUNT(DISTINCT cda.device_token) FILTER (WHERE cda.delivery_state = 'timed-out') AS devices_timed_out,
    COUNT(DISTINCT cda.device_token) FILTER (WHERE cda.delivery_state = 'skipped-sleep-mode') AS devices_skipped_sleep,
    MAX(cda.attempt_number) AS max_retries,
    MIN(cdacks.created_at) FILTER (WHERE cdacks.event = 'push-received') AS first_ack_at,
    EXTRACT(EPOCH FROM (MIN(cdacks.created_at) FILTER (WHERE cdacks.event = 'push-received') - c.created_at)) AS first_ack_latency_sec
FROM calls c
LEFT JOIN call_delivery_attempts cda ON cda.call_id = c.id
LEFT JOIN call_delivery_acks cdacks ON cdacks.call_id = c.id
GROUP BY c.id;
```

#### Management portal additions

Add a "Delivery Health" section to the management portal that shows:

1. **Recent calls** with delivery outcome (answered / unanswered / rejected) and device-level breakdown
2. **Delivery rate** over the last 7 days: % of calls where at least one device acked
3. **Average ack latency** by platform (iOS vs Android)
4. **Failed deliveries** grouped by error type
5. **Apartments with no healthy tokens** (all tokens stale or no tokens registered)

This can be a simple read-only page querying the view above, or API endpoints consumed by the existing management portal.

#### Admin portal additions

Add a "System Delivery Health" section to the admin portal showing cross-building metrics:

1. Delivery rate by building
2. Token health: total tokens, stale tokens pruned in the last 7 days
3. Calls with `delivery-degraded` audit events
4. Retry effectiveness: % of retried devices that eventually acked

#### SQL migration

Create `008-delivery-summary-view.sql` with the view definition above.

### Files To Touch

| File | Change |
|------|--------|
| `vidrom-signaling-server/sql/008-delivery-summary-view.sql` | New migration — delivery summary view |
| `vidrom-signaling-server/lambda/managementRoutes.js` | Add delivery health API endpoints |
| `vidrom-signaling-server/lambda/adminRoutes.js` | Add system-wide delivery health endpoints |
| `vidrom-signaling-server/portals/management.html` | Delivery health UI section |
| `vidrom-signaling-server/portals/admin.html` | System delivery health UI section |

### Verification

- [ ] `call_delivery_summary` view returns correct aggregated data
- [ ] Management portal shows recent calls with device-level delivery breakdown
- [ ] Admin portal shows cross-building delivery rate and token health
- [ ] `delivery-degraded` calls are surfaced prominently
- [ ] Queries perform acceptably on production data volumes

---

## Rollback Safety

All Phase B changes are additive:

- The `call_delivery_attempts` table is new; nothing outside Phase B reads it, so it can be dropped without side effects.
- The `expires_at` column on `calls` is nullable, so existing rows and queries are unaffected.
- Sleep mode filtering is a new check before push; removing it reverts to the current "send to everyone" behavior.
- Retry orchestration uses `setTimeout` in the server process; disabling it returns to single-attempt delivery (Phase A behavior).
- `ring-progress` messages are informational; the intercom ignores unknown message types, so the server can stop sending them without breaking the intercom.
- The delivery summary view is read-only; dropping it doesn't affect writes.
- Portal UI additions are independent pages/sections that can be hidden or removed.

---

## Success Criteria

Phase B is complete when:

1. Ringing sessions persist in the database and survive server restarts.
2. Unacknowledged devices are retried at defined intervals (3s, 8s).
3. The retry orchestrator respects the building's `no_answer_timeout`.
4. `delivery-degraded` events are logged when no device acks after all retries.
5. Sleep mode is checked per-resident before delivery; all-sleeping apartments get an immediate `apartment-unavailable` response.
6. The intercom visitor sees real-time delivery progress (devices notified, confirmed, no response).
7. A delivery summary view and portal UI provide operational visibility into call delivery quality.
8. All changes are additive and can be rolled back by removing the new code/tables.
