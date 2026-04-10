# Step 11-A5 — Device Acknowledgement Events (11.2)

## Problem

The server currently knows only that it *issued* a push request, not whether the device actually received it, woke up, or displayed the incoming call UI. Without positive confirmation, there is no basis for retry decisions (Phase B) or delivery quality monitoring.

## Dependencies

Requires **A4 (Call ID)** — every ack is tied to a `callId`.

## What To Do

### 5a. New SQL migration: `call_delivery_acks`

Create `vidrom-signaling-server/sql/006-call-delivery-acks.sql`:

```sql
CREATE TABLE IF NOT EXISTS call_delivery_acks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    call_id         UUID NOT NULL REFERENCES calls(id) ON DELETE CASCADE,
    user_id         UUID REFERENCES users(id) ON DELETE SET NULL,
    device_token    TEXT NOT NULL,
    token_type      VARCHAR(10) NOT NULL CHECK (token_type IN ('fcm', 'voip')),
    platform        VARCHAR(10) NOT NULL CHECK (platform IN ('ios', 'android')),
    event           VARCHAR(30) NOT NULL CHECK (event IN (
        'push-received', 'app-awake', 'incoming-ui-shown', 'accepted', 'declined'
    )),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_call_delivery_acks_call ON call_delivery_acks (call_id);
CREATE INDEX idx_call_delivery_acks_event ON call_delivery_acks (call_id, event);
```

### 5b. Add HTTP acknowledgement endpoint (`httpRoutes.js`)

```
POST /api/home/calls/:callId/ack
Body: { event, deviceToken, tokenType, platform, userId? }
```

Allowed `event` values:

| Event | When the app sends it |
|-------|-----------------------|
| `push-received` | Background push handler fires (FCM background or VoIP `notification` event) |
| `app-awake` | App resumes or cold-starts because of this call |
| `incoming-ui-shown` | CallKit incoming call UI is displayed (iOS) or full-screen notification shown (Android) |

The endpoint should:

1. Validate `callId` exists in the `calls` table.
2. Insert a row into `call_delivery_acks`.
3. Return `200 OK` with the current call status (so the app knows immediately if the call was already taken or expired).

HTTP is preferred over WebSocket because the WS may not be connected yet when the push handler fires.

### 5c. Home app: send acks at the right moments

| Moment | Where in code | Ack event |
|--------|--------------|-----------|
| VoIP push arrives | `voipService.js` → `notification` listener | `push-received` |
| FCM background message arrives | `fcmService.js` → background handler | `push-received` |
| App comes to foreground for this call | `useCallManager.js` → when `callState` transitions to `incoming` | `app-awake` |
| CallKit UI shown (iOS) | After `reportIncomingCall()` succeeds | `incoming-ui-shown` |
| Android full-screen notification shown | After `showFullScreenCallNotification()` | `incoming-ui-shown` |

Each ack is a fire-and-forget HTTP POST. If it fails, silently ignore — acks are best-effort observability, not required for the call to work.

## Files Touched

| File | Change |
|------|--------|
| `vidrom-signaling-server/sql/006-call-delivery-acks.sql` | New migration |
| `vidrom-signaling-server/src/httpRoutes.js` | New `POST /api/home/calls/:callId/ack` endpoint |
| `vidrom-ai-home/voipService.js` | Send `push-received` ack on VoIP notification |
| `vidrom-ai-home/fcmService.js` | Send `push-received` ack on background FCM message |
| `vidrom-ai-home/useCallManager.js` | Send `app-awake` and `incoming-ui-shown` acks |
| `vidrom-ai-home/callNotification.js` | Send `incoming-ui-shown` ack after showing notification |
| `vidrom-ai-home/config.js` | Add ack endpoint URL helper |

## Verification

- [ ] VoIP push arrives → `push-received` ack row in `call_delivery_acks`
- [ ] FCM background message arrives → `push-received` ack row in `call_delivery_acks`
- [ ] App transitions to incoming call state → `app-awake` ack row
- [ ] CallKit UI shown (iOS) → `incoming-ui-shown` ack row
- [ ] Android full-screen notification shown → `incoming-ui-shown` ack row
- [ ] Ack endpoint returns current call status (e.g. `calling`, `accepted`, `unanswered`)
- [ ] Failed ack POST does not crash the app or block the call flow
- [ ] Ack for a non-existent `callId` returns 404, not a crash
