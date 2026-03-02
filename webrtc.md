# WebRTC Concepts & Vidrom Implementation

## What Is WebRTC?

WebRTC (Web Real-Time Communication) is an open standard that enables **peer-to-peer** audio, video, and data streaming directly between two devices — without routing media through a central server. Once a connection is established the audio/video flows directly between peers, keeping latency low and avoiding server bandwidth costs.

However, *establishing* that direct connection requires several supporting services:

1. **Signaling Service** — exchanges session metadata so both peers know how to connect.
2. **STUN Server** — helps each peer discover its public IP address.
3. **TURN Server** — relays media when a direct path is impossible.

---

## 1. Signaling Service

### Concept

WebRTC does **not** define how peers discover each other. Before any media can flow, both sides must exchange:

| Data | Purpose |
|------|---------|
| **SDP Offer** | Describes the media capabilities (codecs, resolution, etc.) of the caller |
| **SDP Answer** | The callee's matching capabilities |
| **ICE Candidates** | Network endpoint information (IP + port combinations) each peer can be reached at |

A **signaling service** is any transport that relays these messages between the two peers. Common choices are WebSockets, HTTP polling, or push notifications.

### Our Implementation

We run a lightweight **Node.js WebSocket signaling server** (`vidrom-signaling-server`).

- **Intercom device** maintains a **persistent** WebSocket connection with auto-reconnect (always-on SBC).
- **Home app** connects **on-demand** — only when an incoming call arrives (via FCM push) or when the user starts a watch session.
- The server **relays** signaling messages (`offer`, `answer`, `candidate`, `ring`, `accept`, `decline`, `hangup`, `watch`, `watch-end`) between the two peers without inspecting or modifying them.
- It also sends an **FCM push notification** when the intercom rings, so the home app can wake up, connect, and receive the pending ring.
- The server never touches actual audio/video data — media flows peer-to-peer.

**Call flow (simplified):**

```
Intercom                  Signaling Server                  Home App
   │                            │                              │
   │── ring ──────────────────►│                              │
   │                            │── FCM push ────────────────►│
   │                            │◄── WS connect + register ──│
   │                            │── ring (re-sent) ──────────►│
   │                            │◄── accept ──────────────────│
   │◄── accept ─────────────────│                              │
   │                            │◄── offer (SDP) ─────────────│
   │◄── offer (SDP) ────────────│                              │
   │── answer (SDP) ──────────►│                              │
   │                            │── answer (SDP) ────────────►│
   │◄─── ICE candidates ──────►│◄─── ICE candidates ────────►│
   │                            │                              │
   │◄═══════════ direct peer-to-peer media (audio/video) ═══►│
```

**Key files:**
- `vidrom-signaling-server/src/wsHandler.js` — message routing and relay logic
- `vidrom-signaling-server/src/server.js` — HTTP + WebSocket server setup
- `vidrom-signaling-server/src/connectionState.js` — tracks connected clients and pending rings

---

## 2. STUN (Session Traversal Utilities for NAT)

### Concept

Most devices sit behind a NAT router and don't know their own public IP address. A **STUN server** is a lightweight service that a peer queries to discover its **public IP + port** (a "server-reflexive candidate"). Once both peers know their public endpoints, they can attempt a direct connection.

STUN is:
- **Cheap** — just a single round-trip request/response.
- **Stateless** — no ongoing resource usage on the server.
- **Sufficient** in most cases — works whenever both NATs allow hole-punching.

### Our Implementation

Both apps use Google's free public STUN server:

```js
// vidrom-ai-home/config.js  &  vidrom-ai-intercom/webrtcHtml.js
{
  iceServers: [{ urls: 'stun:stun.l.google.com:19302' }]
}
```

The `RTCPeerConnection` is created with this configuration. During ICE gathering, the browser/WebView contacts the STUN server to discover server-reflexive candidates, which are then exchanged via our signaling server.

**Where it is used:**
- **Home app** — `RTCPeerConnection(ICE_SERVERS)` in `vidrom-ai-home/useCallManager.js` (native react-native-webrtc).
- **Intercom app** — `new RTCPeerConnection(ICE_SERVERS)` inside the hidden WebView defined in `vidrom-ai-intercom/webrtcHtml.js` (Chromium WebRTC — chosen to avoid Samsung SIGABRT crashes with the native module).

---

## 3. TURN (Traversal Using Relays around NAT)

### Concept

In some network topologies (symmetric NAT, strict firewalls, carrier-grade NAT), STUN alone cannot establish a direct path. A **TURN server** acts as a media **relay**: both peers send their audio/video to the TURN server, which forwards it to the other side.

TURN is:
- **Always works** — the fallback when direct connection fails.
- **Expensive** — all media bytes flow through the server, consuming bandwidth and adding latency.
- Used as a **last resort** by the ICE framework.

### ICE (Interactive Connectivity Establishment)

ICE is the protocol that ties STUN and TURN together. When a `RTCPeerConnection` is created, ICE gathers **candidates** in priority order:

| Candidate Type | Source | Priority |
|---------------|--------|----------|
| **Host** | Local network interface (LAN IP) | Highest |
| **Server-Reflexive** | Discovered via STUN (public IP) | Medium |
| **Relay** | Allocated on a TURN server | Lowest |

ICE tests connectivity between all candidate pairs and picks the best working path. This means:
- On a LAN (intercom and phone on the same Wi-Fi) → uses **host** candidates for minimal latency.
- Across NATs (e.g., user is away from home) → uses **server-reflexive** candidates via STUN.
- Behind symmetric NAT or firewall → falls back to **relay** candidates via TURN.

### Our Current Status

**We do not currently configure a TURN server.** The ICE configuration only includes a STUN server. This means:

- Calls work well when both devices are on the **same local network** (host candidates).
- Calls work in **most** remote scenarios where NAT hole-punching succeeds (server-reflexive candidates).
- Calls will **fail** in environments with symmetric NAT or strict corporate firewalls where no direct path can be established.

### Adding TURN (Future)

To guarantee connectivity in all network conditions, we would add a TURN server to our ICE configuration:

```js
export const ICE_SERVERS = {
  iceServers: [
    { urls: 'stun:stun.l.google.com:19302' },
    {
      urls: 'turn:turn.vidrom.com:3478',
      username: '<credential>',
      credential: '<password>',
    },
  ],
};
```

Options for provisioning a TURN server:
- **Self-hosted** — deploy [coturn](https://github.com/coturn/coturn) on our AWS infrastructure via CDK.
- **Managed** — use a service like Twilio Network Traversal or Xirsys.

The TURN configuration must be added in **both** apps:
- `vidrom-ai-home/config.js` — `ICE_SERVERS`
- `vidrom-ai-intercom/webrtcHtml.js` — `ICE_SERVERS_JSON`

---

## 4. Architecture Summary

```
┌──────────────────────────────────────────────────────────────────────┐
│                       Vidrom WebRTC Architecture                     │
│                                                                      │
│   ┌─────────────┐         Signaling           ┌─────────────────┐   │
│   │  Intercom    │◄────── WebSocket ──────────►│  Signaling      │   │
│   │  Android SBC │   (offer/answer/ICE/ring)   │  Server (Node)  │   │
│   │              │                              │                 │   │
│   │  WebView     │                              │  + FCM Push     │   │
│   │  (Chromium)  │                              └────────┬────────┘   │
│   └──────┬───────┘                                       │           │
│          │                                               │           │
│          │              ┌──────────────┐          WebSocket (on-demand)
│          │              │  STUN Server │                  │           │
│          │              │  (Google)    │          ┌───────┴────────┐  │
│          │              └──────┬───────┘          │  Home App      │  │
│          │                     │                  │  Android/iOS   │  │
│          │    ◄── IP discovery ──►                │                │  │
│          │                                        │  react-native- │  │
│          │                                        │  webrtc        │  │
│          │◄═══════ Peer-to-Peer Media ══════════►│                │  │
│          │         (audio + video)                 └────────────────┘  │
│                                                                      │
│   ┌─────────────┐  (Future)                                         │
│   │  TURN Server │  Relay fallback when direct connection fails      │
│   └─────────────┘                                                    │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 5. Implementation Notes

### Intercom uses WebView-based WebRTC
The intercom app runs WebRTC inside a **hidden Chromium WebView** instead of using `react-native-webrtc`. This was necessary because Samsung Android SBC devices crash (SIGABRT) with the native module. The WebView approach uses `injectJavaScript` + `postMessage` to bridge signaling messages between React Native and the WebView. See `webrtcHtml.js`.

### Home app uses native WebRTC
The home app uses the `react-native-webrtc` library directly, creating `RTCPeerConnection` instances in JavaScript. See `useCallManager.js` in `vidrom-ai-home`.

### Home is always the offerer
In both call and watch modes, the **home app creates the SDP offer** and the intercom creates the answer. This simplifies the flow — the intercom just needs to respond to offers.

### ICE candidate buffering
Both apps buffer incoming ICE candidates that arrive before the remote description is set, then flush the buffer once the SDP is applied. This handles the common race condition where candidates arrive before the offer/answer exchange completes.

### WebSocket lifecycle
- **Intercom**: persistent connection with auto-reconnect (3-second retry). The device is always on, so maintaining a connection is appropriate.
- **Home**: on-demand connection. Connects when an FCM push arrives or when the user taps "Watch". Disconnects when the session ends to conserve mobile resources.
