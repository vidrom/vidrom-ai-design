# Step 11-A4 — Call ID and Integration with `calls` + `audit_logs` (11.1 + 11.16)

## Problem

The current ring flow is fire-and-forget. There is no `callId` threading messages together, the existing `calls` and `audit_logs` DB tables are never written to, and the server has no durable record of what happened during a ring.

## Dependencies

None for this step itself, but **A5 (acks) and A6 (HTTP accept) both depend on this**.

## What To Do

### 4a. Generate a `callId` on ring

When the server handles a `ring` message:

1. Generate a UUID (`crypto.randomUUID()` or the `gen_random_uuid()` DB default).
2. Insert into the existing `calls` table:
   ```sql
   INSERT INTO calls (id, building_id, apartment_id, intercom_id, status)
   VALUES ($1, $2, $3, $4, 'calling')
   RETURNING id
   ```
   This requires resolving `building_id` and `intercom_id` from the intercom's connection context. The intercom already sends `buildingId` and the server tracks `intercomDeviceId`.
3. Insert into `audit_logs`:
   ```sql
   INSERT INTO audit_logs (id, event_type, building_id, apartment_id, intercom_id, call_id, description)
   VALUES (gen_random_uuid(), 'call-initiated', $1, $2, $3, $4, 'Ring started by intercom')
   ```
4. Store the `callId` in the in-memory active call state (`connectionState.js`).

### 4b. Include `callId` in all messages

Add `callId` to every outbound message that references this ring:

| Message | Direction | Current payload | Add |
|---------|-----------|----------------|-----|
| `ring` | server → home (WS) | `{ type: 'ring' }` | `callId` |
| `incoming-call` | server → home (FCM) | `{ type: 'incoming-call', callerName, apartmentId }` | `callId` |
| `incoming-call` | server → home (VoIP) | `{ callerName, type: 'incoming-call' }` | `callId` |
| `accept` | home → server (WS) | `{ type: 'accept' }` | `callId` |
| `accept` | server → intercom (WS) | `{ type: 'accept' }` | `callId` |
| `decline` | home → server (WS) | `{ type: 'decline' }` | `callId` |
| `call-taken` | server → home (WS) | `{ type: 'call-taken' }` | `callId` |
| `call-taken` | server → home (FCM/VoIP) | `{ type: 'call-taken' }` | `callId` |
| `hangup` | either → server (WS) | `{ type: 'hangup' }` | `callId` |

### 4c. Update `calls` and `audit_logs` on state transitions

| Event | `calls.status` update | `audit_logs` entry |
|-------|----------------------|-------------------|
| Ring started | `calling` (insert) | `call-initiated` |
| First accept | `accepted` | `call-accepted` (include `user_id`) |
| All declined | `rejected` | `call-rejected` |
| Ring expired (timeout) | `unanswered` | `call-unanswered` |
| Hangup (either side) | `ended` + set `ended_at` | `call-ended` |

The pending ring timeout in `connectionState.js` should update the DB when it fires (status → `unanswered`, audit log → `call-unanswered`).

### 4d. Home app: thread `callId` through the call flow

- `useCallManager.js`: when a `ring` WS message or push notification arrives, store the `callId` in state. Include it in `accept`, `decline`, and `hangup` messages sent back to the server.
- `voipService.js`: extract `callId` from VoIP push payload and pass it through.
- `fcmService.js` / `callNotification.js`: extract `callId` from FCM data payload and pass it through.

## Backward Compatibility

All message changes are **additive** — existing fields are preserved, `callId` is a new field. Old home apps that don't send `callId` on accept/decline/hangup will still work — the server treats a missing `callId` as "use the current active call for this apartment" (existing behavior).

### Message Contract Changes

```jsonc
// Server → Home (WS)
{ "type": "ring", "callId": "uuid" }                    // was: { "type": "ring" }
{ "type": "call-taken", "callId": "uuid" }               // was: { "type": "call-taken" }

// Server → Home (FCM data)
{ "type": "incoming-call", "callerName": "Intercom",
  "apartmentId": "uuid", "callId": "uuid" }              // added: callId

// Server → Home (VoIP push)
{ "callerName": "Intercom", "type": "incoming-call",
  "callId": "uuid" }                                     // added: callId

// Home → Server (WS)
{ "type": "accept", "callId": "uuid" }                   // added: callId
{ "type": "decline", "callId": "uuid" }                  // added: callId
{ "type": "hangup", "callId": "uuid" }                   // added: callId
```

## Files Touched

| File | Change |
|------|--------|
| `vidrom-signaling-server/src/wsHandler.js` | Insert into `calls` + `audit_logs` on ring, accept, decline, hangup, timeout. Add `callId` to all outbound messages. |
| `vidrom-signaling-server/src/connectionState.js` | Store `callId` in active call state; update DB on ring expiry |
| `vidrom-ai-home/useCallManager.js` | Thread `callId` through call state and outbound messages |
| `vidrom-ai-home/voipService.js` | Extract `callId` from VoIP push payload |
| `vidrom-ai-home/fcmService.js` | Extract `callId` from FCM data payload |
| `vidrom-ai-home/callNotification.js` | Pass `callId` through notification data |

## Verification

- [ ] Intercom rings → `calls` row created with status `calling`
- [ ] `audit_logs` has `call-initiated` entry with correct `call_id`
- [ ] `callId` present in WS ring message to home
- [ ] `callId` present in FCM push data payload
- [ ] `callId` present in VoIP push payload
- [ ] Home accepts → `calls` updated to `accepted`, `audit_logs` has `call-accepted`
- [ ] Home declines (all residents) → `calls` updated to `rejected`
- [ ] Ring expires → `calls` updated to `unanswered`, `audit_logs` has `call-unanswered`
- [ ] Hangup → `calls` updated to `ended` with `ended_at`, `audit_logs` has `call-ended`
- [ ] Home app sends `callId` in accept, decline, and hangup WS messages
- [ ] Old home app without `callId` support still works (server falls back to active call lookup)
