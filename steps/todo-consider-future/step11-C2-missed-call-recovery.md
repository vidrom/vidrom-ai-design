# C2 — Missed-Call Recovery UX (11.17)

## Context

Part of Step 11 Phase C — Production Hardening.

**Dependencies:** Phase A (callId), Phase B (calls in DB)
**Effort:** Medium

---

## Problem

If a resident's device wakes too late (after the ring expired and no one answered), the app transitions from `incoming` to `idle` with no context. The resident may have seen a brief notification or CallKit UI that vanished, and has no way to know who rang or when.

With `callId` tracking (Phase A) and call history in the `calls` table (Phase A — 11.16), the data exists to show the resident what happened.

## What To Do

### Server — missed-call query endpoint

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

### Server — on ring expiry, push missed-call data over WebSocket

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

### Home app — show missed-call context

**File: `vidrom-ai-home/useCallManager.js`**

Handle the `missed-call` message type:

1. If the app is in `incoming` state (showed the ring but timed out), transition to `idle` and display a brief toast or in-app notification: "Missed call from Main Entrance"
2. If the app is in `idle` state (just connected, ring already expired), show the missed call info as a banner or notification

**File: `vidrom-ai-home/HomeScreen.js`**

Show a missed-call indicator on the home screen:

- Display a "Missed Calls" section or badge when there are recent unanswered calls
- Tapping it could show the list from the `/api/home/calls/missed` endpoint
- Dismiss when the user has seen it

### Home app — on late connect when call is already ended

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

## Files To Touch

| File | Change |
|------|--------|
| `vidrom-signaling-server/src/httpRoutes.js` | Add `GET /api/home/calls/missed` endpoint |
| `vidrom-signaling-server/src/wsHandler.js` | Send `missed-call` on ring expiry; send on late connect if recent unanswered call |
| `vidrom-ai-home/useCallManager.js` | Handle `missed-call` WS message |
| `vidrom-ai-home/HomeScreen.js` | Display missed-call indicator/banner |

## Verification

- [ ] Ring expires → all connected home clients receive `missed-call` with intercom and building context
- [ ] Device that connects shortly after a missed call receives `missed-call` on connect
- [ ] `/api/home/calls/missed` returns recent unanswered calls for the apartment
- [ ] Home screen shows missed-call indicator
- [ ] Missed calls older than 5 minutes are not pushed on connect (only available via API)
- [ ] Calls that were answered by another resident (`call-taken`) are NOT shown as missed
