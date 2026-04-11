# C1 — Late-Join and Extended Ring Lifetime (11.6)

## Context

Part of Step 11 Phase C — Production Hardening.

**Dependencies:** Phase A (callId, building timeout), Phase B (durable ringing)
**Effort:** Small

---

## Problem

A device may wake slowly after a VoIP push — the app launches, connects to the signaling server, but by then the pending ring has already expired. The resident sees a brief flash of the incoming call UI that immediately disappears because the server already moved the call to `unanswered`.

With building-level `no_answer_timeout` (Phase A — 11.13) and durable ringing (Phase B — 11.3), the infrastructure exists to support late-joining devices, but the server currently rejects connections from devices that arrive after the ring timer fires.

## What To Do

### Server — allow late-join on WS connect

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

### Server — allow late-join when call is already accepted elsewhere

If the call has already been accepted (`status = 'accepted'`), the late-arriving device should receive an immediate `call-taken` instead of a stale ring:

```json
{ "type": "call-taken", "callId": "...", "acceptedBy": "..." }
```

This prevents the device from showing an incoming call UI for a call that was already answered.

### Home app — handle late-join gracefully

**File: `vidrom-ai-home/useCallManager.js`**

The existing `ring` handler already sets `callState = 'incoming'` and sends an `app-awake` ack. No changes needed for the base case.

For the `call-taken` case on connect, the existing handler already sets `callState = 'idle'` and cleans up. No changes needed.

### Ring lifetime tuning

The ring lifetime is already driven by the building's `no_answer_timeout` (Phase A — 11.13). No code change needed here, but recommend documenting that building managers can increase this value via the management portal if residents report slow wake times.

A reasonable production default is 30–45 seconds. Buildings with older devices or known slow-wake issues can increase to 60 seconds.

## Files To Touch

| File | Change |
|------|--------|
| `vidrom-signaling-server/src/wsHandler.js` | On home client connect, check for active ringing call and send `ring` or `call-taken` |

## Verification

- [ ] Device that connects during an active ring receives a `ring` message with the call's `callId`
- [ ] Device that connects after call is accepted receives `call-taken` immediately
- [ ] Device that connects after ring expired receives nothing (no stale ring)
- [ ] Late-joining device can accept the call normally (HTTP accept + WS offer flow)
- [ ] `lateJoin: true` is logged for analytics
- [ ] Existing on-time ring behavior is unaffected
