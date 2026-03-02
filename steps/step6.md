# Step 6 - Watch Mode & On-Demand WebSocket Connections

## Goal

Add a "Watch" feature to the home app that lets the user view the intercom's live camera feed without an active call. Also refactor both apps to connect to the signaling server WebSocket only when needed (call or watch session), rather than maintaining a persistent connection.

## Overview

- Home app gets a **Watch** button on the idle screen
- Pressing Watch connects the WebSocket, sends a `watch` request to the server
- Server relays the request to the intercom, which streams video back via WebRTC
- Both apps connect to the WebSocket **on demand** and disconnect when done
- Server tracks pending ring state so the home app doesn't miss calls when woken by FCM

## Prerequisites

1. Steps 1-5 completed (WebRTC calls, FCM push notifications, full-screen incoming call UI)

---

## Step 6.1 - On-Demand WebSocket Architecture

### Rationale

Previously both apps maintained a persistent WebSocket connection with auto-reconnect. This is wasteful and unreliable:
- Android kills background WebSocket connections to save battery
- The intercom only needs a connection when ringing or in a call/watch session
- The home app only needs a connection when responding to a call or initiating a watch session
- FCM push notifications already handle waking the home app for incoming calls

### New Connection Lifecycle

**Intercom App:**
1. Idle → no WebSocket connection
2. User presses Ring → connect WebSocket → register → send `ring` → wait for call flow
3. Call ends → disconnect WebSocket

**Home App:**
1. Idle → no WebSocket connection
2. FCM push received (incoming call) → connect WebSocket → register → server re-sends pending `ring`
3. User presses Watch → connect WebSocket → register → send `watch` request
4. Call/watch session ends → disconnect WebSocket

### Server-Side Pending Ring

When the intercom sends `ring`, the server:
1. Sets `pendingRing = true`
2. Sends FCM push to home app
3. When home app registers via WebSocket, if `pendingRing === true`, re-sends `ring`
4. Clears `pendingRing` on `accept`, `decline`, `hangup`, timeout (30s), or intercom disconnect

---

## Step 6.2 - Refactor Home App `useCallManager`

1. Remove auto-connect on mount and auto-reconnect on disconnect
2. Expose `connectAndRegister()` function that connects the WebSocket + registers as home
3. Expose `disconnect()` function that cleanly closes the WebSocket
4. Call `connectAndRegister()` from:
   - FCM foreground message handler (incoming call)
   - Background notification accept action (app launch)
   - Watch button press
5. Call `disconnect()` when call/watch session ends (hangup, decline, watch-end)

---

## Step 6.3 - Refactor Intercom App `useCallManager`

1. Remove auto-connect on mount and auto-reconnect on disconnect
2. Expose `connectAndRing()` that connects WebSocket → registers → sends `ring`
3. Expose `disconnect()` function
4. Call `connectAndRing()` from the Ring button press
5. Call `disconnect()` when call ends (hangup, decline, timeout)
6. Add `handleWatchRequest()` — when server sends `watch`, start WebRTC video stream (same as call flow but intercom-initiated)

---

## Step 6.4 - Add Watch Support to Signaling Server

1. Handle new message type `watch`:
   ```javascript
   case 'watch': {
     // Home wants to view intercom camera
     if (clients.intercom && clients.intercom.readyState === 1) {
       clients.intercom.send(JSON.stringify({ type: 'watch' }));
     } else {
       ws.send(JSON.stringify({ type: 'error', message: 'Intercom not connected' }));
     }
     break;
   }
   ```

2. Handle `watch-end` message to tell the intercom to stop streaming:
   ```javascript
   case 'watch-end': {
     if (clients.intercom && clients.intercom.readyState === 1) {
       clients.intercom.send(JSON.stringify({ type: 'watch-end' }));
     }
     break;
   }
   ```

3. Add pending ring tracking:
   ```javascript
   let pendingRing = false;
   let pendingRingTimeout = null;

   // In 'ring' handler: set pendingRing = true, start 30s timeout
   // In 'register' handler for home: if pendingRing, re-send ring
   // In 'accept'/'decline'/'hangup': clear pendingRing
   ```

---

## Step 6.5 - Add Watch UI to Home App

1. Add a **Watch** button to `HomeScreen.js` in the idle state:
   ```jsx
   {callState === 'idle' && (
     <View style={styles.centerContent}>
       <Text style={styles.idleText}>No active call</Text>
       <TouchableOpacity style={styles.watchButton} onPress={handleWatch}>
         <Text style={styles.buttonText}>📹 Watch</Text>
       </TouchableOpacity>
     </View>
   )}
   ```

2. Add a `watching` state to the call state machine: `idle | incoming | in-call | watching`

3. In the `watching` state, show the remote video stream with a "Stop Watching" button:
   ```jsx
   {callState === 'watching' && (
     <View style={styles.centerContent}>
       <View style={styles.videoContainer}>
         {remoteStream ? (
           <RTCView streamURL={remoteStream.toURL()} style={styles.remoteVideo} objectFit="cover" />
         ) : (
           <View style={styles.noVideo}>
             <Text style={styles.noVideoText}>Connecting...</Text>
           </View>
         )}
       </View>
       <Text style={styles.callStatus}>Watching</Text>
       <TouchableOpacity style={styles.hangupButton} onPress={handleStopWatch}>
         <Text style={styles.buttonText}>Stop Watching</Text>
       </TouchableOpacity>
     </View>
   )}
   ```

---

## Step 6.6 - Add Watch Handling to Intercom App

1. In `useCallManager.js`, handle the `watch` signaling message:
   - Transition to `in-call` state (reuse the same WebRTC flow)
   - Start camera streaming via WebView

2. Handle `watch-end`:
   - Clean up WebRTC
   - Return to idle
   - Disconnect WebSocket

---

## Testing

1. **Watch flow**: Home idle → press Watch → see intercom video → press Stop → back to idle
2. **Ring flow with on-demand WS**: Intercom ring → home receives FCM → home connects WS → call proceeds normally
3. **Background ring**: Home app killed → intercom rings → FCM wakes home → app opens → WS connects → pending ring delivered → incoming call screen shown
4. **Disconnect cleanup**: After any session ends, verify both apps disconnect their WebSocket within a few seconds
