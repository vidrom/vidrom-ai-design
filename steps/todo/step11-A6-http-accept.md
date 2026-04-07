# Step 11-A6 — Direct HTTP Accept (11.5)

## Problem

When a resident taps "Answer" from CallKit (iOS) or a lock-screen notification (Android), the app must first establish a WebSocket connection, register, and then send an `accept` message. On a cold-started or locked device this can take several seconds — during which another device may win the race, or the ring may expire.

## Dependencies

Requires **A4 (Call ID)** — the HTTP accept is keyed on `callId`.

## What To Do

### 6a. Add HTTP accept endpoint (`httpRoutes.js`)

```
POST /api/home/calls/:callId/accept
Body: { userId, deviceToken }
```

Behavior:

1. Look up the call in the `calls` table. If status is not `calling`, return the current status (e.g., `accepted`, `ended`, `unanswered`) so the app knows immediately.
2. If status is `calling`, atomically update:
   ```sql
   UPDATE calls SET status = 'accepted', updated_at = NOW()
   WHERE id = $1 AND status = 'calling'
   RETURNING *
   ```
   If zero rows affected, the call was already claimed — return current status.
3. Insert `call-accepted` into `audit_logs`.
4. Update in-memory active call state: set `acceptedBy` so the WS-based accept handler won't double-accept.
5. Send `call-taken` to all other home WS clients and push channels (reuse existing call-taken broadcast logic).
6. Notify the intercom WS that the call was accepted.
7. Return `200 OK` with `{ status: 'accepted', callId }`.

The home app then establishes WebSocket and WebRTC normally. The server waits for the WS `register` + offer flow from the accepted device.

### 6b. Home app: call HTTP accept before WS connect

In `useCallManager.js`, when the user taps "Answer" from CallKit or the Android full-screen notification:

1. **Immediately** POST to `/api/home/calls/:callId/accept` (using the `callId` from the push payload).
2. If the response says `accepted`, proceed to connect WS and start WebRTC normally.
3. If the response says `call-taken` or `unanswered` or `ended`, show the appropriate state and don't bother connecting.

This means the call is reserved within milliseconds of the user tapping answer, even if the WS takes several more seconds to establish.

### 6c. Reconcile HTTP accept with WS accept

The existing WS `accept` handler in `wsHandler.js` should check whether the call was already HTTP-accepted by this same device. If so, skip the DB update (it's already done) and proceed directly to the WebRTC offer flow. If a different device HTTP-accepted, send `call-taken`.

## Files Touched

| File | Change |
|------|--------|
| `vidrom-signaling-server/src/httpRoutes.js` | New `POST /api/home/calls/:callId/accept` endpoint |
| `vidrom-signaling-server/src/wsHandler.js` | Reconcile WS accept with prior HTTP accept |
| `vidrom-ai-home/useCallManager.js` | POST HTTP accept before WS connect on answer |
| `vidrom-ai-home/config.js` | Add accept endpoint URL helper |

## Verification

- [ ] HTTP accept for a `calling` call → `calls.status` updated to `accepted`
- [ ] HTTP accept returns `{ status: 'accepted', callId }`
- [ ] HTTP accept when call already accepted by another device → returns `{ status: 'call-taken' }`
- [ ] HTTP accept when call expired → returns `{ status: 'unanswered' }`
- [ ] HTTP accept triggers `call-taken` WS + push to all other home devices
- [ ] HTTP accept notifies the intercom WS that the call was accepted
- [ ] Subsequent WS accept from the same device doesn't double-accept (recognizes prior HTTP accept)
- [ ] Subsequent WS accept from a different device gets `call-taken`
- [ ] Home app on lock-screen: tap answer → HTTP accept fires within milliseconds, before WS connects
- [ ] Home app proceeds to WS + WebRTC setup after successful HTTP accept
- [ ] Home app shows "call taken" UI if HTTP accept returns `call-taken`
