# Channels & Notification Rules

## Session Types

The system supports two session types between the intercom and home apps:

| Session Type | Initiator | Direction | Home sends audio | Intercom UI |
|-------------|-----------|-----------|-----------------|-------------|
| **Call** | Intercom (ring) | Intercom → Home | Yes (mic) | InCallScreen |
| **Watch** | Home app (watch) | Home → Intercom | No (recvonly) | Stays on normal screen |

---

## Single-Session Rule (Per Intercom)

Only **one active session per intercom** exists at a time. The server tracks each session in an `activeCalls` Map keyed by intercom `deviceId`, with a `type` field (`'call'` or `'watch'`).

---

## Priority Rules

**Calls always take priority over watch sessions.**

| Scenario | Result |
|----------|--------|
| Home A starts watch (no active session) | Watch starts normally |
| Home B starts watch while Home A is watching | Home A gets `watch-end`, Home B takes over |
| Intercom rings while someone is watching | Watch ends (`watch-end` sent to watcher), call proceeds |
| Home tries to watch during an active call | Rejected with error: "Intercom is busy on a call" |
| Home A accepts a call while Home B is watching | N/A — ring already ended the watch |

### Summary

- **Call hijacks watch** — always.
- **Watch does NOT hijack call** — never.
- **Watch hijacks watch** — last watcher wins, previous watcher is cleanly disconnected.

---

## Watch Mode Behavior on the Intercom

During a watch session the intercom app **stays on its normal screen** (home / apartment list / door code). The WebView for WebRTC is mounted at the App level in the background so the camera streams without navigating to InCallScreen.

When the intercom initiates a call (ring), any active watch session is cleaned up first — the WebRTC peer connection is closed and a new one is created for the call.

---

## Notification Flow (Calls Only)

Watch sessions do **not** trigger push notifications. Only intercom-initiated calls do:

```
Intercom App ── WS: { type: 'ring', apartmentId } ──→ Signaling Server
                                                        ├── WS: { type: 'ring' } to all connected home clients for that apartment
                                                        ├── FCM push (Android + iOS foreground)
                                                        └── VoIP push via APNs (iOS — triggers CallKit full-screen UI)
```

### Push Token Types

| Token Type | Platform | Registered via | Triggers |
|-----------|----------|----------------|----------|
| **FCM** | Android, iOS | `fcmService.js` → WS `register-fcm-token` | Data-only push with `type: 'incoming-call'` |
| **VoIP (PushKit)** | iOS only | `voipService.js` → WS `register-voip-token` | APNs VoIP push → CallKit incoming call screen |

### First-Accept-Wins

When multiple residents of the same apartment are notified:

1. All connected home clients receive `{ type: 'ring' }` via WebSocket
2. All registered devices receive push notifications
3. The **first** resident to accept wins — server stores their WebSocket as `acceptedWs`
4. All other residents receive `{ type: 'call-taken' }`
5. If **all** connected residents decline, the server relays `{ type: 'decline' }` to intercom

---

## WebRTC Session Lifecycle

Both call and watch sessions use the same WebRTC flow once established:

1. **Home** creates `RTCPeerConnection`, creates SDP offer, sends to server
2. **Server** relays offer to intercom
3. **Intercom** WebView creates `RTCPeerConnection`, gets camera+mic, creates SDP answer
4. ICE candidates exchanged via server relay
5. Direct peer-to-peer media established

### Cleanup

- **Call ends**: `hangup` or `open-door` message → both sides tear down peer connection
- **Watch ends**: `watch-end` message → both sides tear down peer connection
- **Disconnect**: `peer-disconnected` sent to the other side → cleanup
- **ICE failure**: WebView detects `disconnected`/`failed` state → cleanup

---

## Multi-Tenant Architecture

The signaling server supports **multiple buildings simultaneously** on a single instance. Each building has its own intercom device, and sessions are fully isolated between buildings.

### Server-Side State

| Data Structure | Key | Value | Purpose |
|---------------|-----|-------|---------|
| `intercoms` Map | `deviceId` | `{ ws, buildingId }` | All connected intercom devices |
| `activeCalls` Map | `deviceId` | `{ apartmentId, type, acceptedBy, ... }` | Active sessions per intercom |
| `homeClients` Map | `wsKey` | `{ ws, apartmentId, buildingId }` | All connected home clients |

### Building Resolution

Home apps register with `apartmentId`. The server resolves the building and intercom at connection time:

```
Home connects ── WS: { type: 'register', role: 'home', apartmentId } ──→ Server
                                                                          ├── DB: SELECT building_id FROM apartments WHERE id = apartmentId
                                                                          ├── Lookup: getIntercomForBuilding(buildingId)
                                                                          └── Stores: { ws, apartmentId, buildingId } in homeClients
```

Intercoms register with their `deviceId` and `buildingId` (from JWT):

```
Intercom connects ── WS: { type: 'register', role: 'intercom' } ──→ Server
                                                                      ├── JWT: { deviceId, buildingId }
                                                                      └── Stores: { ws, buildingId } in intercoms Map
```

### Building Isolation

- A ring from Building A's intercom only notifies home clients in Building A
- A call in Building A does not block or affect calls in Building B
- Watch sessions target the specific intercom for the watcher's building
- Each intercom has its own independent `activeCall` entry in the `activeCalls` Map

### Message Routing

All signaling messages (offer, answer, ICE candidates, hangup, open-door) are routed through the resolved intercom for the home client's building:

```
Home (Bldg A) ── offer ──→ Server ── getIntercom(intercomDeviceId) ──→ Intercom A
Home (Bldg B) ── offer ──→ Server ── getIntercom(intercomDeviceId) ──→ Intercom B
```

### Legacy Backward Compatibility

For intercom devices that haven't been re-provisioned, the server falls back to the old `clients.intercom` singleton. This ensures existing deployments continue working until intercoms are updated.
