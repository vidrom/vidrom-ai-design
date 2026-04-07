# Step 11-A1 — WebSocket Heartbeat & Max-Call-Duration Safety Net

## Problem

1. **Zombie connections** — The intercom maintains a persistent WebSocket to the signaling server, but neither side sends periodic pings. A zombie TCP connection can appear open for minutes before the OS raises an error. During that window, the server believes the intercom is connected, but any ring sent over the dead socket is silently lost.

2. **Stuck calls** — If both sides crash mid-call, or a hangup message is lost, the active-call slot stays occupied indefinitely. No cleanup ever fires because neither side explicitly disconnects.

## Dependencies

None — standalone, no prerequisites.

## What Was Done

### 1. Server-side ping/pong heartbeat (`server.js` + `wsHandler.js`)

- Added a `setInterval` (every 20s) on the WSS that iterates all connected sockets and calls `ws.ping()`.
- Each socket gets `isAlive = true` on connection and on every `pong` event.
- If `isAlive` is still `false` when the interval fires, the socket is terminated → triggers the existing `close` handler which updates `intercoms.status = 'disconnected'` in the DB.

### 2. Server-side max-call-duration timeout (`wsHandler.js`)

- A 60s timer starts when a call is **accepted** (first-accept-wins).
- On expiry, both sides receive `{ type: 'hangup', reason: 'timeout' }`, and the active-call slot + pending ring are cleared.
- The timer is cleared on normal hangup, open-door, decline (all declined), and disconnect — so it only fires as a safety net.

### 3. Intercom client-side ping watchdog (`useCallManager.js`)

- A 45s "no-data watchdog" resets on every `onmessage` and `onopen`.
- If no data arrives for 45s (server pings every 20s, so this means 2+ missed pings), the WebSocket is closed and the existing auto-reconnect kicks in after 3s.
- Watchdog is cleaned up on `onclose` and component unmount.

## Files Touched

| File | Change |
|------|--------|
| `vidrom-signaling-server/src/server.js` | Added 20s ping interval on WSS with `isAlive` tracking |
| `vidrom-signaling-server/src/wsHandler.js` | `ws.isAlive = true` + `pong` listener on connect; 60s max-call-duration timer (start on accept, clear on any call end) |
| `vidrom-ai-intercom/useCallManager.js` | 45s no-data watchdog → close + auto-reconnect |

## Verification

- [x] Server sends WS ping every ~20s to all connected sockets
- [x] Server terminates sockets that miss a pong within one interval
- [x] Terminated intercom socket triggers `intercoms.status = 'disconnected'` DB update
- [x] Calls auto-hangup after 60s with `reason: 'timeout'` sent to both sides
- [x] Timer is cleared on normal hangup/open-door/decline/disconnect
- [x] Intercom detects no-data after ~45s and reconnects proactively
- [x] Normal connections with good network are unaffected
