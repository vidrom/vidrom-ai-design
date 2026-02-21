# Step 2 - Real-Time Communication Between Intercom and Home Apps

## Goal

Establish a signaling server and WebRTC-based audio/video calling between vidrom-intercom (door unit) and vidrom-home (resident's phone), with a basic "ring" → accept/decline call flow.

## Overview

- A lightweight Node.js + WebSocket signaling server relays call events and WebRTC session descriptions between the two apps
- Both apps use `react-native-webrtc` for peer-to-peer audio/video
- vidrom-intercom initiates a "ring" to vidrom-home
- vidrom-home shows an incoming-call UI; the user accepts or declines
- On accept, a two-way audio+video WebRTC session is established

## Components

1. **vidrom-signaling-server** – Node.js signaling server (new project)
2. **vidrom-intercom** – updated Expo app (door-side)
3. **vidrom-home** – updated Expo app (resident-side)

## Step 2.1 - Create the Signaling Server

1. Create a new folder `vidrom-signaling-server/` in the repo root
2. Initialize: `npm init -y`
3. Install dependencies: `npm install ws uuid`
4. Implement `server.js`:
   - Start a WebSocket server on port 8080
   - Track connected clients by role ("intercom" / "home")
   - Relay the following message types between peers:
     - `"register"` – client registers its role
     - `"ring"` – intercom notifies home of incoming call
     - `"accept"` – home accepts the call
     - `"decline"` – home declines the call
     - `"offer"` – WebRTC SDP offer
     - `"answer"` – WebRTC SDP answer
     - `"candidate"` – ICE candidate exchange
     - `"hangup"` – either side ends the call

5. Test: start the server and verify two WebSocket clients can connect and exchange messages

## Step 2.2 - Add WebRTC to vidrom-intercom

1. Install react-native-webrtc:
   ```bash
   npx expo install react-native-webrtc
   ```
   
   **Note:** This requires a development build (Expo Go does not support native modules). Create a dev build with:
   ```bash
   npx expo prebuild
   npx expo run:android
   ```

2. Connect to the signaling server via WebSocket on app start
3. Register with role "intercom"
4. Update UI:
   - Show local camera preview (RTCView)
   - Add a "Ring" button that sends a "ring" message to the server
   - On receiving "accept":
     - Create RTCPeerConnection
     - Add local audio+video tracks
     - Create and send SDP offer via signaling server
     - Handle incoming SDP answer
     - Exchange ICE candidates
     - Display remote audio stream (home user's mic)
   - On receiving "decline": show "Call Declined" message briefly
   - Add a "Hang Up" button to end an active call

5. Handle cleanup: close peer connection and media tracks on hangup

## Step 2.3 - Add WebRTC to vidrom-home

1. Install react-native-webrtc (same as above, requires dev build)
2. Connect to the signaling server via WebSocket on app start
3. Register with role "home"
4. Update UI:
   - Default screen: "Waiting..." or idle state
   - On receiving "ring":
     - Show an incoming-call screen with:
       - "Incoming Call from Intercom" label
       - "Accept" button
       - "Decline" button
   - On accept:
     - Send "accept" to signaling server
     - Wait for SDP offer from intercom
     - Create RTCPeerConnection
     - Add local audio track (mic)
     - Set remote description (offer), create and send SDP answer
     - Exchange ICE candidates
     - Display remote video stream (intercom camera) in RTCView
   - On decline: send "decline" to signaling server, return to idle
   - Add a "Hang Up" button to end an active call

5. Handle cleanup: close peer connection and media tracks on hangup

## Step 2.4 - Verify Project Structure

After completion, the workspace should look like:

```
vidrom-ai/
├── design/
│   ├── spec.md
│   ├── step1.md
│   ├── step1-steps.txt
│   └── step2.md
├── vidrom-signaling-server/   <-- NEW: signaling server
│   ├── server.js
│   ├── package.json
│   └── package-lock.json
├── vidrom-intercom/            <-- Updated with WebRTC + ring flow
│   ├── App.js
│   ├── app.json
│   ├── package.json
│   └── ...
└── vidrom-home/                <-- Updated with WebRTC + call UI
    ├── App.js
    ├── app.json
    ├── package.json
    └── ...
```

## Step 2.5 - End-to-End Test

1. Start the signaling server:
   ```bash
   cd vidrom-signaling-server && node server.js
   ```

2. Run vidrom-intercom on an Android device (dev build):
   ```bash
   cd vidrom-intercom && npx expo run:android
   ```

3. Run vidrom-home on a second device (dev build):
   ```bash
   cd vidrom-home && npx expo run:android
   # or for iOS (requires macOS):
   # npx expo run:ios
   ```

4. Test flow:
   - Both apps connect to the signaling server (verify in server logs)
   - Press "Ring" on vidrom-intercom
   - vidrom-home shows incoming call UI
   - Press "Accept" on vidrom-home
   - Verify two-way audio and one-way video (intercom camera → home)
   - Press "Hang Up" on either side and verify call ends cleanly
   - Press "Ring" again, then "Decline" — verify call is rejected

## Completion Criteria

- ✅ Signaling server runs and correctly relays messages
- ✅ vidrom-intercom can ring vidrom-home
- ✅ vidrom-home can accept or decline incoming calls
- ✅ On accept: WebRTC audio+video session established (intercom camera streams to home; two-way audio)
- ✅ Hang up works from either side
- ✅ No crashes or resource leaks after multiple call cycles

## Notes

- For local testing, both devices must be on the same Wi-Fi network and the signaling server IP must be reachable from both
- TURN/STUN servers: for local network testing, a public STUN server (e.g., `stun:stun.l.google.com:19302`) is sufficient. TURN will be needed later for NAT traversal across different networks
- Full-screen incoming-call notifications (like WhatsApp) will be addressed in a later step
